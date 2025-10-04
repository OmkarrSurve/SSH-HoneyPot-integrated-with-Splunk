# Cowrie
Cowrie is a medium-interaction SSH and Telnet honeypot designed to emulate a real server environment and log all attacker activity. It records login attempts, keystrokes, and commands executed by an attacker, while also capturing files downloaded via wget or scp. By simulating a vulnerable SSH service, Cowrie helps security researchers and administrators collect threat intelligence, analyze attack techniques, and monitor brute-force attempts in a controlled environment without exposing the actual system


# Cowrie Setup  

- Follow these steps on your Linux VM.
- Note: **$ signs are not part of the commands**

## 1) Install prerequisites (run as sudo-enabled user/root)
```bash
$ sudo apt-get install git python3-pip python3-venv libssl-dev libffi-dev build-essential libpython3-dev python3-minimal authbind

```


## 2) Create a user account
```bash
$ sudo adduser --disabled-password cowrie
#Output Section
Adding user 'cowrie' ...
Adding new group 'cowrie' (1002) ...
Adding new user 'cowrie' (1002) with group 'cowrie' ...
Changing the user information for cowrie
Enter the new value, or press ENTER for the default
Full Name []:      #here you can type cowrie to give name and click enter, for everything below click enter and at last type y and click enter
Room Number []:
Work Phone []:
Home Phone []:
Other []:
Is the information correct? [Y/n]
#Output Section

$ sudo su - cowrie
```


## 3) Checkout the Code
```bash
$ git clone http://github.com/cowrie/cowrie
$ cd cowrie
```


## 4) Setup Virtual Environment 

### Create your virtual environment
```bash
$ pwd
$ python3 -m venv cowrie-env
```

### Activate the virtual environment and install packages
```bash
$ source cowrie-env/bin/activate
(cowrie-env) $ python -m pip install --upgrade pip
(cowrie-env) $ python -m pip install -e .
```


## 5) Installing and Modifying Configuration file
```bash
(cowrie-env) $ cd etc/
(cowrie-env) $ ls         #In Output you will see a file named cowrie.cfg.dist
(cowrie-env) $ cp cowrie.cfg.dist cowrie.cfg     #Creates a copy of the file
(cowrie-env) $ nano cowrie.cfg     #For editing the file
#Changes
# [telnet]
# enabled = True

# hostname = ssh_server (you can name anything)
```


## 6) Listening on Port 22
## Iptables
### Port redirection commands are system-wide and need to be executed as root. A firewall redirect can make your existing SSH server unreachable, remember to move the existing server to a different port number first
### The following firewall rule will forward incoming traffic on port 22 to port 2222 on Linux:
```bash
$ sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
```
### For Telnet
```bash
$ sudo iptables -t nat -A PREROUTING -p tcp --dport 23 -j REDIRECT --to-port 2223
```



## 7) Starting Cowrie
```bash
$ source cowrie-env/bin/activate
(cowrie-env) $ cowrie start
```

## 8) To Stop Cowrie
```bash
(cowrie-env) $ cowrie stop
```

























