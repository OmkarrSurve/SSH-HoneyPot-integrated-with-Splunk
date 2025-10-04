# Splunk HEC Setup for Cowrie Honeypot

##Splunk HTTP Event Collector (HEC) provides a lightweight, secure, and real-time method to send Cowrie honeypot logs from Linux VM to Splunk on Windows. Instead of installing a full Splunk Universal Forwarder or moving raw log files, the Python forwarder streams each JSON event directly over HTTPS using a token for authentication, ensuring cross-platform compatibility, reduced overhead, and instant visibility of SSH attack activity in Splunk for analysis and dashboards.

## Part A — On Windows Splunk: enable HEC & create a token

1. Open Splunk Web on Windows log in as admin.

2. Enable HEC:
   - Go to **Settings → Data inputs → HTTP Event Collector**.
   - Click **Global Settings** (top right) → Enable HEC and set Port to `8088` (default). Save.

3. Create a new HEC token:
   - Click **New Token**.
   - Name: `cowrie_hec`
   - Source type: (choose `cowrie` or `json`), Index: create or select `cowrie`.
   - Click **Save** and copy the Token value (long string) to use on your Linux VM.

4. Configure Windows Firewall:
   - Click win+R, type:control
   - Go to **Windows Defender Firewall**
   - Click **Advanced settings** on the left panel.This opens Windows Defender Firewall with Advanced Security.
   - In the left pane, click **Inbound Rules**.
   - In the right pane, click **New Rule…**
   - Choose **Port → Next.**
   - Select TCP and enter 8088 in **“Specific local ports” → Next.**
   - Select **Allow the connection → Next.**
   - Choose Domain, Private, Public (all checked unless you want to restrict) → Next.
   - Give the rule a name, e.g., **Splunk HEC → Finish.**
   - This Allow inbound TCP port `8088` for the Splunk server host so Kali can reach it and optionally restricts source IPs to your Linux VM IP.

6. Confirm Splunk is listening (on Windows cmd/powershell):
   ```
   netstat -an | findstr 8088
   ```
   — it should show `LISTENING`.

## Part B — On your Linux VM: allow cowrie outbound to Windows Splunk

Note: 
   - If your VM is on the same host or device, ensure your windows ip is not the localhost one.
   - To get your windows ip, open command prompt and type ipconfig, this will provide you your windows ip configuration
   - Choose the ethernet adapter ethernet 3 IP (Worked for me)

```bash
# run as root or sudo
WINDOWS_IP=<WINDOWS_IP>    # replace with your Windows machine IP
COWRIE_UID=$(id -u cowrie)

# allow cowrie user to reach Splunk HEC port 8088
sudo iptables -I OUTPUT -m owner --uid-owner $COWRIE_UID -d $WINDOWS_IP -p tcp --dport 8088 -j ACCEPT

# ensure previous DROP rule is still present and remains last (if you used it)
sudo iptables -C OUTPUT -m owner --uid-owner $COWRIE_UID -j DROP 2>/dev/null || sudo iptables -I OUTPUT -m owner --uid-owner $COWRIE_UID -j DROP

# Save rules for persistence (Debian/Kali)
sudo netfilter-persistent save
```


## Part C — Quick test with curl 

Before wiring the forwarder, test HEC works from your VM:

```bash
WINDOWS_IP=<WINDOWS_IP>
HEC_PORT=8088
HEC_TOKEN=<PASTE_YOUR_TOKEN>

# single event test
curl -k https://$WINDOWS_IP:$HEC_PORT/services/collector/event \
  -H "Authorization: Splunk $HEC_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"event":"hello from cowrie test","sourcetype":"cowrie","index":"cowrie"}'
```

If successful, you should see:
```json
{"text":"Success","code":0}
```
and the event will appear in Splunk search: `index=cowrie sourcetype=cowrie "hello from cowrie test"`.

## Part D — Create a Lightweight Python forwarder 

Create this file in your Linux VM's Desktop, save it as a py file
```python
#!/usr/bin/env python3
"""
cowrie_hec_forwarder.py
Tails cowrie JSON logfile and posts each line to Splunk HEC.
Run as cowrie user inside the cowrie venv.
"""

import time, os, requests, sys, json, logging
from pathlib import Path

# CONFIG - edit these
HEC_URL = "https://<WINDOWS_IP>:8088/services/collector"
HEC_TOKEN = "<YOUR_HEC_TOKEN>"
COWRIE_JSON = "/home/cowrie/cowrie/var/log/cowrie/cowrie.json"
SOURCETYPE = "cowrie"
INDEX = "cowrie"
BATCH_SIZE = 1         #Batch size is one because I conducted a small attack of only 4-5 events,
                       #if batch size is large, it will take time to load the events in splunk or they will not appear too, 
                       #so either conduct 50-60 events or decrease batch size          
SLEEP_ON_EMPTY = 1.0  # seconds   
# setup logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")

session = requests.Session()
session.verify = False  # set to True if you have valid certs
session.headers.update({"Authorization": f"Splunk {HEC_TOKEN}", "Content-Type": "application/json"})

def send_event(event_json):
    payload = {"event": event_json, "sourcetype": SOURCETYPE, "index": INDEX}
    try:
        r = session.post(HEC_URL, json=payload, timeout=10)
        if r.status_code != 200:
            logging.warning("HEC returned %s: %s", r.status_code, r.text.strip()[:200])
            return False
        return True
    except Exception as e:
        logging.warning("HEC post error: %s", e)
        return False

def tail_file(path):
    p = Path(path)
    if not p.exists():
        logging.error("File not found: %s", path)
        sys.exit(1)
    with open(path, "r", encoding="utf-8", errors="ignore") as fh:
        fh.seek(0, os.SEEK_END)
        while True:
            line = fh.readline()
            if not line:
                time.sleep(SLEEP_ON_EMPTY)
                continue
            yield line.rstrip("\n")

def main():
    buffer = []
    for raw in tail_file(COWRIE_JSON):
        raw = raw.strip()
        if not raw:
            continue
        try:
            obj = json.loads(raw)
        except Exception:
            obj = {"raw": raw}
        buffer.append(obj)
        if len(buffer) >= BATCH_SIZE:
            for ev in buffer:
                ok = send_event(ev)
                if not ok:
                    logging.warning("Failed to send event; will retry after sleep")
                    time.sleep(5)
                    break
            buffer.clear()

if __name__ == "__main__":
    try:
        logging.info("Starting Cowrie->Splunk HEC forwarder")
        main()
    except KeyboardInterrupt:
        logging.info("Stopping forwarder")
        sys.exit(0)
```

### Saving and Running the file

   - Start the normal terminal

```bash
cd /Desktop
ls         #If the file is created it will appear in the output
sudo mv ~/Desktop/cowrie_hec_forwarder.py /home/cowrie/cowrie/     #move it to the cowrie venv 
sudo chown cowrie2:cowrie2 /home/cowrie2/cowrie/cowrie_hec_forwarder.py     #Fix ownership
```


   - Run inside cowrie venv
```bash
su - cowrie
cd /home/cowrie/cowrie
source cowrie-env/bin/activate
python3 /home/cowrie/cowrie_hec_forwarder.py &   # run in background for quick test
```

Check Splunk for incoming events:
```spl
index=cowrie sourcetype=cowrie | head 20
```


