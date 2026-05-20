# Port 7 - Echo

## Table of Contents

- [Enumeration](#enumeration)
- [Exploitation](#exploitation)

---

## Enumeration

### Quick Check (One-liner)

```shell
echo "test" | nc $rhost 7 && echo "Echo service responds"
```

### Banner Grabbing

```shell
nc -vn $rhost 7
```

### Nmap Scripts

```shell
nmap -sV -sC -p7 $rhost
```

### UDP Echo

```shell
nmap -sU -p7 $rhost
```

---

## Exploitation

### Testing Echo Service

```shell
# TCP
echo "test" | nc $rhost 7

# UDP
echo "test" | nc -u $rhost 7
```

### Amplification Attack (DDoS)

> Echo service can be used in amplification attacks

```shell
# Check if echo responds (potential amplification)
hping3 -c 1 --udp -p 7 $rhost
```

### Network Reconnaissance

```shell
# Use echo to verify host is alive
for ip in $(seq 1 254); do
  echo "test" | nc -w1 192.168.1.$ip 7 2>/dev/null && echo "192.168.1.$ip is alive"
done
```

---

## References

- [HackTricks - Echo](https://book.hacktricks.wiki/network-services-pentesting/7-tcp-udp-pentesting-echo.html)
