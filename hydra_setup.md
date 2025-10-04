# Hydra Setup for Simulated Events on Cowrie Honeypot
Hydra is a fast, flexible password-cracking tool used to perform controlled brute-force and credential-stuffing tests against authentication services (like SSH); in this project we use Hydra only to simulate attacker behavior against the local Cowrie honeypot so you can validate logging, detection, and alerting. It lets you run small, configurable username/password lists with adjustable concurrency and delays to emulate real attack patterns, but must be used responsibly — only against systems you own or have explicit permission to test — to avoid legal and ethical issues

 - Open the terminal of your Linux VM
 - Install Hydra
```bash
sudo apt update && sudo apt install -y hydra
```

 - Make tiny lists for events
```bash
# make tiny test lists
cat > /tmp/hydra-users.txt <<'EOF'
root
admin
guest
test
EOF

cat > /tmp/hydra-pass.txt <<'EOF'
toor
admin123
password
1234
EOF
```
 - Run Hydra
```bash
# run hydra against local cowrie on 2222 (gentle)
hydra -L /tmp/hydra-users.txt -P /tmp/hydra-pass.txt -s 2222 -t 2 -w 3 -f -V ssh://127.0.0.1

```
