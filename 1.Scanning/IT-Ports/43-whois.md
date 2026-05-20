# Port 43 - WHOIS

## Table of Contents

- [Enumeration](#enumeration)
- [Information Gathering](#information-gathering)
- [Exploitation](#exploitation)

---

## Enumeration

### Quick Check (One-liner)

```shell
whois $domain && whois -h $rhost $domain
```

### Banner Grabbing

```shell
nc -vn $rhost 43
```

### Nmap Scripts

```shell
nmap -sV -sC -p43 $rhost
```

---

## Information Gathering

### Basic WHOIS Query

```shell
whois $domain
whois $ip
```

### Query Specific WHOIS Server

```shell
whois -h $rhost $domain
whois -h whois.verisign-grs.com $domain
```

### Regional Internet Registries

```shell
# ARIN (North America)
whois -h whois.arin.net $ip

# RIPE (Europe, Middle East, Central Asia)
whois -h whois.ripe.net $ip

# APNIC (Asia Pacific)
whois -h whois.apnic.net $ip

# LACNIC (Latin America)
whois -h whois.lacnic.net $ip

# AFRINIC (Africa)
whois -h whois.afrinic.net $ip
```

### Extract Useful Information

```shell
# Get registrar info
whois $domain | grep -i "registrar"

# Get nameservers
whois $domain | grep -i "name server"

# Get admin contact
whois $domain | grep -i "admin"

# Get creation/expiry dates
whois $domain | grep -i "date"
```

---

## Exploitation

### WHOIS Injection

> Some WHOIS servers may be vulnerable to command injection

```shell
# Test for injection
whois -h $rhost '$(id)'
whois -h $rhost '; ls -la'
```

### Information Disclosure

```shell
# Gather email addresses for phishing
whois $domain | grep -i "@"

# Find related domains (same registrant)
whois $domain | grep -i "registrant"
```

### Reverse WHOIS

```shell
# Find other domains owned by same entity
# Use online tools: viewdns.info, domaintools.com
```

---

## Online Tools

- ViewDNS: https://viewdns.info/whois/
- DomainTools: https://whois.domaintools.com/
- ICANN Lookup: https://lookup.icann.org/

---

## References

- [HackTricks - WHOIS](https://book.hacktricks.wiki/network-services-pentesting/43-pentesting-whois.html)
