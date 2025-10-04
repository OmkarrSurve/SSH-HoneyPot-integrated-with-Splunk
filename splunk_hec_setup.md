# Splunk HEC Setup for Cowrie Honeypot

## Part A — On Windows Splunk: enable HEC & create a token

1. Open Splunk Web on Windows: `http://<WINDOWS_IP>:8000` and log in as admin.

2. Enable HEC:
   - Go to **Settings → Data inputs → HTTP Event Collector**.
   - Click **Global Settings** (top right) → Enable HEC and set Port to `8088` (default). Save.

3. Create a new HEC token:
   - Click **New Token**.
   - Name: `cowrie_hec`
   - Source type: (choose `cowrie` or `json`), Index: create or select `cowrie`.
   - Click **Save** and copy the Token value (long string) to use on Kali.

4. Configure Windows Firewall:
   - Allow inbound TCP port `8088` for the Splunk server host so Kali can reach it.
   - Optionally restrict source IPs to your Kali VM IP.

5. Confirm Splunk is listening (on Windows cmd/powershell):
   ```
   netstat -an | findstr 8088
   ```
   — it should show `LISTENING`.

## Part B — On Kali: allow cowrie outbound to Windows Splunk

If you previously blocked all outbound traffic for the cowrie user, allow only the Splunk IP:

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

> If you didn’t block outbound earlier, ignore this step.

## Part C — Quick test with curl (one-off)

Before wiring the forwarder, test HEC works from Kali:

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

## Part D — Lightweight Python forwarder (recommended)

Save the following as `/home/cowrie/cowrie_hec_forwarder.py` (owner: cowrie) and run inside the cowrie venv. It tails `cowrie.json` and posts each JSON event to HEC.

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
BATCH_SIZE = 50
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

### Usage (on Kali, as cowrie user)

```bash
# put file in /home/cowrie/, set owner and permissions
chown cowrie:cowrie /home/cowrie/cowrie_hec_forwarder.py
chmod 750 /home/cowrie/cowrie_hec_forwarder.py

# run inside cowrie venv
su - cowrie
cd /home/cowrie/cowrie
source cowrie-env/bin/activate
python3 /home/cowrie/cowrie_hec_forwarder.py &   # run in background for quick test
```

Check Splunk for incoming events:
```spl
index=cowrie sourcetype=cowrie | head 20
```

> If you want the forwarder to keep running across reboots, create a systemd service.

