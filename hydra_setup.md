# Hydra Setup for Simulated Events on Cowrie Honeypot

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