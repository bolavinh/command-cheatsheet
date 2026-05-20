# Port 49 - TACACS+

## Table of Contents

- [Enumeration](#enumeration)
- [Exploitation](#exploitation)
- [Decryption](#decryption)

---

## Enumeration

### Quick Check (One-liner)

```shell
nmap -sV -p 49 --version-intensity 5 $rhost
```

### Nmap Scripts

```shell
nmap -sV -sC -p49 $rhost
```

### Banner Grabbing

```shell
nc -vn $rhost 49
```

### Service Detection

```shell
nmap -sV -p49 --version-intensity 5 $rhost
```

---

## Exploitation

### Brute Force

> TACACS+ uses shared secret key for encryption

```shell
# Using Loki (TACACS+ testing tool)
# https://github.com/c0decafe/loki

# Install
git clone https://github.com/c0decafe/loki.git
cd loki
pip install -r requirements.txt

# Brute force shared secret
python loki.py -i $rhost -w /usr/share/wordlists/rockyou.txt
```

### MITM Attack

> Intercept TACACS+ traffic and crack shared secret

```shell
# Capture TACACS+ traffic with tcpdump
tcpdump -i eth0 -w tacacs.pcap port 49

# Extract and crack with tacacsplus
# https://github.com/carlospolop/hacktricks/blob/master/generic-methodologies-and-resources/shells/linux.md
```

---

## Decryption

### Crack TACACS+ Shared Secret

```shell
# Using tacacs_crack.py
# Requires captured TACACS+ packet

# Extract packet data from pcap
tshark -r tacacs.pcap -T fields -e data > tacacs_data.txt

# Brute force with custom script
# https://github.com/c0decafe/loki
```

### Wireshark Decryption

```
1. Edit > Preferences > Protocols > TACACS
2. Enter shared secret key
3. Decrypt captured traffic
```

---

## Configuration Files

### Cisco TACACS+ Configuration

```
# Common config locations
/etc/tacacs+/tac_plus.conf
/etc/tac_plus.conf

# Cisco device config
tacacs-server host $rhost key $secret
```

### Key Information to Look For

- Shared secret keys
- User credentials
- Authorization rules
- Accounting data

---

## Tools

- Loki: https://github.com/c0decafe/loki
- tacacs_crack: https://github.com/moparisthebest/tacacs_crack
- Wireshark (with TACACS+ dissector)

---

## References

- [HackTricks - TACACS+](https://book.hacktricks.wiki/network-services-pentesting/49-pentesting-tacacs+.html)
