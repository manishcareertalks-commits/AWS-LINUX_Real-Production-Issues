# Top 20 Linux Commands — DevOps Engineer Cheatsheet

A practical reference for DevOps, Cloud, SRE, and Linux engineers. Each command includes what it does, syntax, useful options, examples, expected output, and common interview/production use cases.

---

## 1. `ps aux` — View Running Processes

### What it does

`ps` displays information about running processes. The `aux` options show processes for all users with detailed CPU and memory information.

### Syntax

```bash
ps aux
```

### Useful options

```bash
ps aux
ps -ef
ps aux | grep nginx
ps aux --sort=-%cpu
ps aux --sort=-%mem
```

### Example

```bash
ps aux | grep nginx
```

Example output:

```text
root      1234  0.0  1.2  ... nginx: master process
www-data  1250  0.1  2.1  ... nginx: worker process
```

### Important columns

| Column | Meaning |
|---|---|
| USER | User owning the process |
| PID | Process ID |
| %CPU | CPU utilization |
| %MEM | Memory utilization |
| VSZ | Virtual memory size |
| RSS | Resident memory size |
| STAT | Process state |
| START | Process start time |
| COMMAND | Command that started the process |

### DevOps use case

If an EC2 server is experiencing high CPU, start with:

```bash
ps aux --sort=-%cpu | head
```

To identify memory-heavy processes:

```bash
ps aux --sort=-%mem | head
```

---

## 2. `top` — Real-Time Process Monitoring

### What it does

`top` provides a continuously updating view of CPU, memory, load average, and running processes.

### Syntax

```bash
top
```

### Useful interactive keys

| Key | Action |
|---|---|
| `P` | Sort by CPU |
| `M` | Sort by memory |
| `1` | Show individual CPU cores |
| `k` | Kill a process |
| `r` | Change process priority |
| `q` | Quit |

### Example

```bash
top
```

Example:

```text
top - 10:21:30 up 15 days, 2 users
%Cpu(s): 1.3 us, 0.7 sy, 0.0 ni, 97.6 id
MiB Mem : 3866.8 total, 1250.6 free
```

### DevOps use case

When a Linux server suddenly becomes slow:

```bash
top
```

Check:

1. CPU utilization
2. Load average
3. Memory usage
4. High-CPU processes
5. High-memory processes

For a more convenient alternative on many systems:

```bash
htop
```

---

## 3. `df -h` — Check Disk Space

### What it does

`df` reports available and used disk space for mounted filesystems.

The `-h` option displays values in human-readable units such as GB and MB.

### Syntax

```bash
df -h
```

### Example

```bash
df -h
```

Example output:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1       40G   15G   23G  40% /
/dev/xvdb       100G   92G  3.2G  97% /var
```

### Useful commands

Check a particular mount:

```bash
df -h /var
```

Check filesystem type:

```bash
df -Th
```

Check inode usage:

```bash
df -ih
```

### DevOps use case

If an application returns errors because the server cannot write files:

```bash
df -h
```

If disk space appears available but files still cannot be created, check inodes:

```bash
df -ih
```

---

## 4. `du -sh /path` — Find Directory Size

### What it does

`du` estimates file and directory space usage.

### Syntax

```bash
du -sh /path
```

### Common examples

```bash
du -sh /var/log
```

Check sizes of items inside a directory:

```bash
du -sh /var/log/*
```

Find the largest directories:

```bash
du -h --max-depth=1 /var | sort -hr
```

### Options

| Option | Meaning |
|---|---|
| `-s` | Summary |
| `-h` | Human-readable |
| `-a` | Include files |
| `-d` | Limit directory depth |

### DevOps use case

If `/var` is almost full:

```bash
du -sh /var/*
```

You might discover:

```text
8.5G  /var/log
3.2G  /var/lib
1.1G  /var/cache
```

Then investigate the largest directory.

---

## 5. `ss -tulpn` — Check Listening Ports

### What it does

`ss` displays socket statistics and is commonly used to identify listening TCP/UDP ports and the processes using them.

### Syntax

```bash
ss -tulpn
```

### Option meanings

| Option | Meaning |
|---|---|
| `-t` | TCP |
| `-u` | UDP |
| `-l` | Listening |
| `-p` | Show process |
| `-n` | Do not resolve service names |

### Example

```bash
ss -tulpn
```

Example output:

```text
Netid State  Local Address:Port  PID/Program
tcp   LISTEN 0.0.0.0:22          1234/sshd
tcp   LISTEN 0.0.0.0:80          2345/nginx
```

### Check a specific port

```bash
ss -tulpn | grep :80
```

### DevOps use case

If Nginx should be listening on port 80 but the application is unreachable:

```bash
ss -tulpn | grep :80
```

This helps determine whether a service is actually listening.

---

## 6. `ls -lah` — List Files with Details

### What it does

`ls` lists files and directories. `-l` provides detailed information, `-a` includes hidden files, and `-h` makes sizes human-readable.

### Syntax

```bash
ls -lah
```

### Example

```bash
ls -lah /var/log
```

Example:

```text
drwxr-xr-x  2 root root 4.0K May 20 10:10 .
-rw-r--r--  1 root root 1.2K May 20 10:10 messages
```

### Important fields

```text
permissions owner group size date filename
```

### Useful commands

```bash
ls -la
ls -lh
ls -lt
ls -ltr
ls -lah /etc
```

`ls -lt` is particularly useful for finding recently modified files.

---

## 7. `find` — Search for Files

### What it does

`find` recursively searches directories based on conditions such as name, type, size, owner, permissions, and modification time.

### Syntax

```bash
find /path -name "pattern"
```

### Example

```bash
find /var -name "error.log"
```

Example output:

```text
/var/log/error.log
/var/www/html/error.log
```

### Common examples

Find all `.log` files:

```bash
find /var/log -name "*.log"
```

Case-insensitive search:

```bash
find /var -iname "error.log"
```

Find directories:

```bash
find /var -type d -name "backup"
```

Find files larger than 1 GB:

```bash
find /var -type f -size +1G
```

Find files modified in the last 24 hours:

```bash
find /var/log -type f -mtime -1
```

Find and delete files carefully:

```bash
find /tmp -type f -name "*.tmp" -delete
```

### DevOps use case

When troubleshooting a server, `find` is useful for locating:

- Application logs
- Configuration files
- Large files
- Recently modified files
- Core dumps
- Temporary files

---

## 8. `cat` — Display File Contents

### What it does

`cat` prints file contents to standard output.

### Syntax

```bash
cat filename
```

### Example

```bash
cat /etc/hostname
```

Possible output:

```text
ip-10-0-0-12
```

### Useful examples

Display multiple files:

```bash
cat file1.txt file2.txt
```

Show line numbers:

```bash
cat -n file.txt
```

Create a small file:

```bash
cat > config.txt
```

Append content:

```bash
cat >> config.txt
```

### When to use it

`cat` is best for small files.

For very large logs, prefer:

```bash
less /var/log/syslog
```

---

## 9. `less` — View Large Files Page by Page

### What it does

`less` lets you inspect large files without loading the entire file into a terminal window at once.

### Syntax

```bash
less filename
```

### Example

```bash
less /var/log/syslog
```

### Useful keys

| Key | Action |
|---|---|
| `Space` | Next page |
| `b` | Previous page |
| `/pattern` | Search |
| `n` | Next search result |
| `N` | Previous search result |
| `g` | Beginning |
| `G` | End |
| `q` | Quit |

### Example

Search for errors:

```text
/error
```

Then press `n` to move between matches.

### DevOps use case

Use `less` for large application and system logs where `cat` would flood the terminal.

---

## 10. `cp` — Copy Files and Directories

### What it does

`cp` copies files or directories.

### Syntax

```bash
cp source destination
```

### Examples

Copy a file:

```bash
cp file.txt /backup/
```

Rename through copying:

```bash
cp file.txt file_backup.txt
```

Copy a directory recursively:

```bash
cp -r application/ /backup/application/
```

Preserve permissions and timestamps:

```bash
cp -p file.txt /backup/
```

Copy an entire directory while preserving attributes:

```bash
cp -a application/ /backup/application/
```

### DevOps use case

Before changing a configuration file:

```bash
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```

This gives you a quick rollback copy.

---

## 11. `mv` — Move or Rename Files

### What it does

`mv` moves files/directories or renames them.

### Syntax

```bash
mv source destination
```

### Rename a file

```bash
mv file.txt file_old.txt
```

### Move a file

```bash
mv app.log /backup/
```

### Move a directory

```bash
mv application /opt/
```

### DevOps use case

Rotate or archive a log:

```bash
mv application.log application.log.old
```

Then restart or reload the application so a new log can be created.

---

## 12. `rm` — Remove Files or Directories

### What it does

`rm` removes files. With recursive options it can remove directories and their contents.

### Syntax

```bash
rm file.txt
```

### Examples

Remove a file:

```bash
rm file.txt
```

Prompt before removal:

```bash
rm -i file.txt
```

Remove an empty directory:

```bash
rmdir directory
```

Remove a directory recursively:

```bash
rm -r directory
```

Force removal:

```bash
rm -rf directory
```

### Important warning

Be extremely careful with:

```bash
rm -rf
```

A mistaken path can delete important data.

Always verify the target first:

```bash
pwd
ls -lah
```

---

## 13. `useradd` — Create a Linux User

### What it does

`useradd` creates a new user account.

### Syntax

```bash
sudo useradd username
```

### Example

```bash
sudo useradd devops
```

Set a password:

```bash
sudo passwd devops
```

Create a user with a home directory and Bash shell:

```bash
sudo useradd -m -s /bin/bash devops
```

Add the user to a group:

```bash
sudo usermod -aG docker devops
```

### Verify the user

```bash
id devops
```

```bash
getent passwd devops
```

### DevOps use case

Create dedicated service or administrative accounts rather than sharing the root account.

---

## 14. `chmod` — Change File Permissions

### What it does

`chmod` changes read, write, and execute permissions.

Linux permissions are commonly represented as:

```text
r = 4
w = 2
x = 1
```

For example:

```text
7 = rwx
6 = rw-
5 = r-x
4 = r--
```

### Syntax

```bash
chmod permissions file
```

### Example

```bash
chmod 755 script.sh
```

This means:

```text
Owner  = rwx = 7
Group  = r-x = 5
Others = r-x = 5
```

### Common permissions

```bash
chmod 644 file.txt
chmod 755 script.sh
chmod 600 private.key
chmod 700 private_directory
```

### Symbolic examples

```bash
chmod u+x script.sh
chmod g+w file.txt
chmod o-r file.txt
```

### DevOps use case

A deployment script may need execute permission:

```bash
chmod +x deploy.sh
```

---

## 15. `chown` — Change File Owner and Group

### What it does

`chown` changes the owner and/or group of files and directories.

### Syntax

```bash
chown user:group file
```

### Example

```bash
chown ubuntu:ubuntu file.txt
```

Change ownership recursively:

```bash
sudo chown -R ubuntu:ubuntu /var/www/app
```

Change only owner:

```bash
sudo chown ubuntu file.txt
```

Change only group:

```bash
sudo chown :developers file.txt
```

### Verify ownership

```bash
ls -l file.txt
```

### DevOps use case

A web application may fail to write files because the application user does not own the directory:

```bash
sudo chown -R www-data:www-data /var/www/html
```

Always confirm the correct service account before changing ownership.

---

## 16. `tar -czf` — Create a Compressed Archive

### What it does

`tar` packages multiple files/directories into one archive. With `-z`, gzip compression is used.

### Syntax

```bash
tar -czf archive.tar.gz /path
```

### Example

```bash
tar -czf backup.tar.gz /var/www
```

### Option meanings

| Option | Meaning |
|---|---|
| `-c` | Create archive |
| `-x` | Extract archive |
| `-z` | gzip compression |
| `-v` | Verbose |
| `-f` | Archive filename |

### Extract

```bash
tar -xzf backup.tar.gz
```

Extract to a directory:

```bash
tar -xzf backup.tar.gz -C /backup/
```

List archive contents:

```bash
tar -tzf backup.tar.gz
```

### Other useful formats

Create an uncompressed tar:

```bash
tar -cf backup.tar /var/www
```

Create gzip archive with verbose output:

```bash
tar -czvf backup.tar.gz /var/www
```

### DevOps use case

Before a major deployment:

```bash
tar -czf app-backup.tar.gz /var/www/app
```

---

## 17. `wget` — Download Files

### What it does

`wget` downloads files from HTTP, HTTPS, and other supported protocols.

### Syntax

```bash
wget URL
```

### Example

```bash
wget https://example.com/file.zip
```

### Download with a custom filename

```bash
wget -O application.zip https://example.com/file.zip
```

### Continue an interrupted download

```bash
wget -c https://example.com/file.zip
```

### Download in the background

```bash
wget -b https://example.com/file.zip
```

### DevOps use case

Download a release artifact, package, script, or configuration file from a trusted server.

---

## 18. `curl` — Transfer Data to/from Servers

### What it does

`curl` is widely used to make HTTP requests and transfer data.

### Syntax

```bash
curl URL
```

### Check HTTP headers

```bash
curl -I https://example.com
```

Example:

```text
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
```

### GET request

```bash
curl https://example.com
```

### Follow redirects

```bash
curl -L https://example.com
```

### POST JSON data

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"devops"}' \
  https://api.example.com/users
```

### Test a local application

```bash
curl http://localhost:8080/health
```

### DevOps use case

`curl` is extremely useful for checking:

- Load balancers
- REST APIs
- Application health endpoints
- Reverse proxies
- HTTP status codes
- Network connectivity

For example:

```bash
curl -I https://myapplication.example.com
```

---

## 19. `journalctl -xe` — Troubleshoot System Logs

### What it does

`journalctl` queries logs collected by `systemd-journald`.

### Syntax

```bash
journalctl
```

The commonly used troubleshooting command from the cheatsheet is:

```bash
journalctl -xe
```

### Useful commands

View recent logs:

```bash
journalctl -xe
```

View logs for a service:

```bash
journalctl -u nginx
```

View recent logs for a service:

```bash
journalctl -u nginx -n 100
```

Follow logs in real time:

```bash
journalctl -u nginx -f
```

View logs since a time:

```bash
journalctl --since "1 hour ago"
```

View logs from the current boot:

```bash
journalctl -b
```

### DevOps use case

If a systemd service fails:

```bash
systemctl status nginx
journalctl -u nginx -xe
```

This combination often reveals configuration errors, permission problems, missing files, or failed dependencies.

---

## 20. `uptime` — Check System Uptime and Load

### What it does

`uptime` shows how long the system has been running, the number of logged-in users, and the load average.

### Syntax

```bash
uptime
```

Example:

```text
10:21:30 up 15 days, 2 users, load average: 0.15, 0.10, 0.05
```

### Load average

The three values represent approximately:

```text
1 minute   5 minutes   15 minutes
```

For example:

```text
load average: 0.15, 0.10, 0.05
```

A high load average relative to the number of CPU cores can indicate CPU contention or processes waiting for resources.

Check CPU count:

```bash
nproc
```

### DevOps use case

During a performance incident:

```bash
uptime
nproc
top
```

Use these together to understand system load and CPU capacity.

---

# Essential Linux Troubleshooting Workflow

The 20 commands become much more useful when combined during an incident.

## Scenario 1 — Server is Slow

Start with:

```bash
uptime
top
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
```

Then check disk:

```bash
df -h
```

Check for large directories:

```bash
du -sh /var/* | sort -hr
```

---

## Scenario 2 — Application Is Not Reachable

Check whether the process exists:

```bash
ps aux | grep application
```

Check listening ports:

```bash
ss -tulpn
```

Check the service:

```bash
systemctl status application
```

Check logs:

```bash
journalctl -u application -xe
```

Test locally:

```bash
curl http://localhost:8080/health
```

---

## Scenario 3 — Disk Is Full

Check filesystem usage:

```bash
df -h
```

Check inode usage:

```bash
df -ih
```

Find large directories:

```bash
du -sh /var/* | sort -hr
```

Find large files:

```bash
find /var -type f -size +1G -ls
```

Inspect logs:

```bash
du -sh /var/log/*
```

---

## Scenario 4 — Permission Denied

Inspect the file:

```bash
ls -lah file.txt
```

Check ownership:

```bash
ls -l file.txt
```

Check current user:

```bash
whoami
```

Check groups:

```bash
id
```

Modify permissions if appropriate:

```bash
chmod 644 file.txt
```

Modify ownership if appropriate:

```bash
sudo chown user:group file.txt
```

Do not blindly use `chmod 777`; fix the actual ownership or permission requirement.

---

## Scenario 5 — Nginx Is Not Working

Check the process:

```bash
ps aux | grep nginx
```

Check port 80:

```bash
ss -tulpn | grep :80
```

Check service status:

```bash
systemctl status nginx
```

Check logs:

```bash
journalctl -u nginx -xe
```

Test configuration:

```bash
nginx -t
```

Test HTTP response:

```bash
curl -I http://localhost
```

---

# Linux Permissions Quick Reference

```text
Permission   Numeric value
---------    -------------
r            4
w            2
x            1
```

Common combinations:

```text
7 = rwx
6 = rw-
5 = r-x
4 = r--
0 = ---
```

Common permission modes:

```bash
chmod 644 file.txt
chmod 755 script.sh
chmod 600 private.key
chmod 700 private_directory
```

Remember:

```text
chmod 755 file
     │││
     ││└── Others
     │└─── Group
     └──── Owner
```

---

# Linux File Ownership

A typical `ls -l` result:

```text
-rwxr-xr-- 1 ubuntu developers 1200 May 20 10:10 script.sh
```

The important ownership fields are:

```text
ubuntu      = owner
developers  = group
```

Useful commands:

```bash
whoami
id
ls -l
chown
chgrp
```

---

# Useful Command Combinations for DevOps Engineers

## Find the process consuming the most CPU

```bash
ps aux --sort=-%cpu | head -10
```

## Find the process consuming the most memory

```bash
ps aux --sort=-%mem | head -10
```

## Find top directories by disk usage

```bash
du -h --max-depth=1 / | sort -hr | head
```

## Find large files

```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

## Check whether port 8080 is listening

```bash
ss -tulpn | grep :8080
```

## Find an application process

```bash
ps aux | grep myapp
```

## Follow application logs

```bash
journalctl -u myapp -f
```

## Test an application endpoint

```bash
curl -I http://localhost:8080
```

## Find recently modified files

```bash
find /var/log -type f -mtime -1
```

## Archive a directory

```bash
tar -czf backup.tar.gz /var/www/app
```

---

# Important DevOps Interview Scenarios

## 1. EC2 CPU is 100%. What commands do you run?

```bash
uptime
top
ps aux --sort=-%cpu | head
```

Then identify the process and investigate its logs/configuration.

---

## 2. EC2 disk usage is 97%. How do you troubleshoot?

```bash
df -h
du -sh /var/* | sort -hr
find /var -type f -size +1G -ls
```

Then inspect log files and application-generated files.

---

## 3. Application is running but users cannot access it.

Check:

```bash
ps aux | grep application
ss -tulpn
curl http://localhost:8080
```

If the service is managed by systemd:

```bash
systemctl status application
journalctl -u application -xe
```

Then investigate firewall, security group, load balancer, DNS, and application configuration.

---

## 4. Nginx returns 502 Bad Gateway.

Start with:

```bash
systemctl status nginx
ss -tulpn
journalctl -u nginx -xe
curl http://localhost:BACKEND_PORT
```

The objective is to determine whether Nginx is healthy and whether its upstream application is reachable.

---

## 5. A deployment script returns "Permission denied".

Check:

```bash
ls -lah deploy.sh
```

Check whether it is executable:

```bash
chmod +x deploy.sh
```

Check ownership:

```bash
ls -l deploy.sh
```

Check the executing user:

```bash
whoami
```

---

## 6. A service suddenly stopped after a reboot.

Check:

```bash
systemctl status service-name
journalctl -u service-name -b
```

The `-b` option limits logs to the current boot, which makes post-reboot troubleshooting easier.

---

# Quick Tips

- Use `Tab` for shell auto-completion.
- Use `man <command>` for detailed command documentation.
- Use `Ctrl + C` to stop a running foreground command.
- Linux is case-sensitive.
- Always verify destructive commands before executing them.
- Avoid `rm -rf` unless you fully understand the target path.
- Avoid `chmod 777` as a default fix; use least privilege.
- Use `sudo` only when elevated privileges are required.
- Prefer `less` for large files and logs.
- Combine commands with pipes (`|`) to build powerful troubleshooting workflows.

---

# Command Cheat Sheet

| # | Command | Primary Use |
|---:|---|---|
| 1 | `ps aux` | View running processes |
| 2 | `top` | Real-time process monitoring |
| 3 | `df -h` | Check filesystem disk usage |
| 4 | `du -sh /path` | Check directory size |
| 5 | `ss -tulpn` | Check listening ports |
| 6 | `ls -lah` | List files with details |
| 7 | `find` | Search for files/directories |
| 8 | `cat` | Display file contents |
| 9 | `less` | View large files page by page |
| 10 | `cp` | Copy files/directories |
| 11 | `mv` | Move/rename files |
| 12 | `rm` | Remove files/directories |
| 13 | `useradd` | Create users |
| 14 | `chmod` | Change permissions |
| 15 | `chown` | Change ownership |
| 16 | `tar -czf` | Create compressed archives |
| 17 | `wget` | Download files |
| 18 | `curl` | Transfer/test HTTP data |
| 19 | `journalctl -xe` | Troubleshoot systemd logs |
| 20 | `uptime` | Check uptime and load |

---

# Final Takeaway

For DevOps engineers, memorizing commands is less important than knowing **which command to use during a real production problem**.

A useful mental model is:

```text
PROCESS PROBLEM
    ↓
ps / top

DISK PROBLEM
    ↓
df / du / find

NETWORK / PORT PROBLEM
    ↓
ss / curl

FILE PROBLEM
    ↓
ls / cat / less / find

PERMISSION PROBLEM
    ↓
ls / chmod / chown

SERVICE PROBLEM
    ↓
systemctl / journalctl

BACKUP / ARCHIVE
    ↓
tar / cp

SYSTEM HEALTH
    ↓
uptime / top
```

These commands form a strong Linux troubleshooting foundation for DevOps, Cloud, SRE, and System Administrator interviews and day-to-day production work.
