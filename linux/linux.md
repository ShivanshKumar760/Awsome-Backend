# Linux Commands for Application Deployment

A complete reference of Linux commands you'll need when deploying and managing applications on a server.

---

## 1. System Updates & Package Management

```bash
# Update package list
sudo apt update

# Upgrade all installed packages
sudo apt upgrade -y

# Update + upgrade in one line
sudo apt update && sudo apt upgrade -y

# Install a package
sudo apt install -y <package-name>

# Remove a package
sudo apt remove <package-name>

# Remove package and its config files
sudo apt purge <package-name>

# Remove unused dependencies
sudo apt autoremove -y

# Search for a package
apt search <keyword>

# Show package info
apt show <package-name>

# List installed packages
dpkg --list

# Check if a specific package is installed
dpkg -l | grep <package-name>
```

---

## 2. File & Directory Operations

```bash
# List files (detailed)
ls -la

# List files sorted by modified time (newest first)
ls -lt

# Change directory
cd /path/to/directory

# Go to home directory
cd ~

# Go back one directory
cd ..

# Print current directory
pwd

# Create a directory
mkdir my-app

# Create nested directories
mkdir -p /home/deploy/apps/api

# Remove a file
rm filename.txt

# Remove a directory and all contents
rm -rf /path/to/directory

# Copy a file
cp source.txt destination.txt

# Copy a directory recursively
cp -r source-dir/ dest-dir/

# Move or rename a file/directory
mv old-name new-name

# Create an empty file
touch newfile.txt

# Show file contents
cat filename.txt

# Show file contents with line numbers
cat -n filename.txt

# View large files page by page
less filename.txt

# Show first 20 lines
head -n 20 filename.txt

# Show last 20 lines
tail -n 20 filename.txt

# Follow log file in real time
tail -f /var/log/app.log

# Find files by name
find /home/deploy -name "*.jar"

# Find files modified in the last 24 hours
find . -mtime -1

# Search text inside files
grep -r "search-term" /path/to/dir

# Search case-insensitively
grep -ri "error" /var/log/

# Show disk usage of a directory (human-readable)
du -sh /home/deploy/

# Show disk usage per subfolder
du -sh /home/deploy/*/

# Check available disk space
df -h

# Check available disk space for a specific path
df -h /home
```

---

## 3. File Permissions & Ownership

```bash
# View permissions
ls -la

# Make a file executable
chmod +x script.sh

# Set specific permissions (owner=rwx, group=rx, others=r)
chmod 754 script.sh

# Common permission values
# 777 — everyone can read, write, execute  (avoid for security)
# 755 — owner full, others read+execute    (good for scripts/dirs)
# 644 — owner read+write, others read      (good for config files)
# 600 — owner read+write only              (good for .env / keys)

# Change file owner
sudo chown deploy filename.txt

# Change owner and group
sudo chown deploy:deploy filename.txt

# Change ownership recursively
sudo chown -R deploy:deploy /home/deploy/my-app/

# Change group only
sudo chgrp deploy filename.txt
```

---

## 4. User & Group Management

```bash
# Create a new user
sudo adduser deploy

# Create a user without interactive prompts
sudo useradd -m -s /bin/bash deploy

# Set or change password
sudo passwd deploy

# Add user to a group (e.g., sudo or docker)
sudo usermod -aG sudo deploy
sudo usermod -aG docker deploy

# Switch to another user
su - deploy

# Switch to root
sudo -i

# Run a single command as another user
sudo -u deploy command

# Show current user
whoami

# Show all groups the current user belongs to
groups

# Show all users on the system
cat /etc/passwd

# Delete a user (keep home directory)
sudo userdel deploy

# Delete a user and their home directory
sudo userdel -r deploy

# List all groups
cat /etc/group
```

---

## 5. SSH & Remote Access

```bash
# Connect to a remote server
ssh user@SERVER_IP

# Connect on a custom SSH port
ssh -p 2222 user@SERVER_IP

# Connect using a specific private key
ssh -i ~/.ssh/my-key.pem user@SERVER_IP

# Copy SSH key to a remote server (enables passwordless login)
ssh-copy-id user@SERVER_IP

# Generate a new SSH key pair
ssh-keygen -t ed25519 -C "your-email@example.com"

# Copy files from local to remote
scp localfile.txt user@SERVER_IP:/remote/path/

# Copy files from remote to local
scp user@SERVER_IP:/remote/file.txt ./local/path/

# Copy a directory recursively
scp -r ./my-app/ user@SERVER_IP:/home/deploy/

# Sync files (only changed files) — efficient for re-deploys
rsync -avz ./my-app/ user@SERVER_IP:/home/deploy/my-app/

# SSH tunnel — forward remote port to local
ssh -L 5433:localhost:5432 user@SERVER_IP -N

# Keep SSH session alive
ssh -o ServerAliveInterval=60 user@SERVER_IP
```

---

## 6. Process Management

```bash
# List all running processes
ps aux

# Filter processes by name
ps aux | grep node

# Show real-time process usage (CPU + RAM)
top

# Better process viewer (install if needed)
sudo apt install -y htop
htop

# Find the PID of a process by name
pgrep node
pgrep java

# Kill a process by PID
kill 1234

# Force kill a process
kill -9 1234

# Kill all processes by name
pkill node
pkill java

# Show what process is using a specific port
sudo ss -tlnp | grep :3000
sudo lsof -i :3000

# Show all open ports
sudo ss -tlnp

# Run a process in the background
nohup java -jar app.jar &

# Bring background process to foreground
fg

# List background jobs
jobs

# Disown a background job (keeps it running after SSH disconnect)
disown -h %1
```

---

## 7. systemd — Service Management

```bash
# Start a service
sudo systemctl start my-app

# Stop a service
sudo systemctl stop my-app

# Restart a service
sudo systemctl restart my-app

# Reload config without full restart (if the service supports it)
sudo systemctl reload my-app

# Enable service to start on boot
sudo systemctl enable my-app

# Disable service from starting on boot
sudo systemctl disable my-app

# Check if a service is running
sudo systemctl status my-app

# Check if a service is enabled
sudo systemctl is-enabled my-app

# Reload systemd after editing a .service file
sudo systemctl daemon-reload

# List all active services
sudo systemctl list-units --type=service --state=active

# List all services (active + inactive)
sudo systemctl list-units --type=service

# View service logs (live)
journalctl -u my-app -f

# View last 50 lines of service logs
journalctl -u my-app -n 50

# View logs since last boot
journalctl -u my-app -b

# View logs for a time range
journalctl -u my-app --since "2026-06-01" --until "2026-06-16"
```

**Template: Create a systemd service file**

```bash
sudo nano /etc/systemd/system/my-app.service
```

```ini
[Unit]
Description=My Application
After=network.target

[Service]
User=deploy
WorkingDirectory=/home/deploy/my-app
ExecStart=/usr/bin/command-to-start-app
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable my-app
sudo systemctl start my-app
```

---

## 8. Firewall — UFW

```bash
# Check firewall status and rules
sudo ufw status
sudo ufw status numbered
sudo ufw status verbose

# Enable the firewall
sudo ufw enable

# Disable the firewall
sudo ufw disable

# Allow a port
sudo ufw allow 3000
sudo ufw allow 8080/tcp

# Allow a named service
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw allow 'Nginx HTTP'
sudo ufw allow 'Nginx HTTPS'

# Allow a port from a specific IP only
sudo ufw allow from 203.0.113.45 to any port 5432

# Allow an IP range
sudo ufw allow from 192.168.1.0/24

# Deny a port
sudo ufw deny 5432
sudo ufw deny 3306

# Delete a rule by number
sudo ufw status numbered
sudo ufw delete 3

# Delete a rule by specification
sudo ufw delete allow 3000

# Reset all firewall rules
sudo ufw reset

# Reload firewall rules
sudo ufw reload
```

---

## 9. Environment Variables

```bash
# View all environment variables
printenv

# View a specific variable
echo $HOME
echo $PATH
echo $USER

# Set a temporary variable (current session only)
export APP_PORT=3000

# Set a permanent variable for the current user
echo 'export APP_PORT=3000' >> ~/.bashrc
source ~/.bashrc

# Set system-wide environment variables
sudo nano /etc/environment
# Add: APP_PORT=3000
# Then log out and back in to apply

# Create a .env file for an app
nano /home/deploy/my-app/.env
# Contents:
# DB_HOST=localhost
# DB_PORT=5432
# DB_USER=myuser
# DB_PASS=mysecretpassword

# Read .env file in a script
export $(grep -v '^#' .env | xargs)
```

---

## 10. Networking

```bash
# Show IP addresses
ip addr show
ip a

# Show network interfaces
ip link show

# Check connectivity to a host
ping google.com
ping -c 4 google.com       # limit to 4 packets

# Trace network path to a host
traceroute google.com

# DNS lookup
nslookup your-domain.com
dig your-domain.com

# Check if a port is open on a remote server
nc -zv SERVER_IP 5432
telnet SERVER_IP 3000

# Show all listening ports
sudo ss -tlnp
sudo netstat -tlnp          # older alternative

# Show active connections
ss -tp

# Get your public IP address
curl https://ipinfo.io/ip

# Download a file
curl -O https://example.com/file.zip
wget https://example.com/file.zip

# Make an API request
curl http://localhost:3000/api/health
curl -X POST http://localhost:5000/api/data \
  -H "Content-Type: application/json" \
  -d '{"key":"value"}'
```

---

## 11. Log Management

```bash
# Common log locations
/var/log/syslog          # General system logs
/var/log/auth.log        # SSH and authentication logs
/var/log/nginx/          # Nginx access and error logs
/var/log/apt/            # Package installation logs

# View Nginx access log
sudo tail -f /var/log/nginx/access.log

# View Nginx error log
sudo tail -f /var/log/nginx/error.log

# Search logs for errors
sudo grep -i "error" /var/log/nginx/error.log

# Search logs for a specific IP
sudo grep "203.0.113.45" /var/log/nginx/access.log

# View system logs
sudo journalctl -xe

# View kernel logs
sudo dmesg | tail -50

# Clear old logs (older than 7 days)
sudo journalctl --vacuum-time=7d

# Limit journal log size
sudo journalctl --vacuum-size=500M

# Rotate logs manually
sudo logrotate -f /etc/logrotate.conf
```

---

## 12. Disk & Memory Usage

```bash
# Check disk space (all mounts)
df -h

# Check disk space for a specific directory
df -h /home

# Check directory size
du -sh /home/deploy/

# Check sizes of all subdirectories
du -sh /home/deploy/*/

# Find the largest files in a directory
du -ah /home/deploy/ | sort -rh | head -20

# Check RAM usage
free -h

# Check RAM + swap usage
free -m

# Real-time memory and CPU stats
vmstat 2         # updates every 2 seconds

# Check CPU info
lscpu

# Check how many CPU cores
nproc

# Check server uptime and load average
uptime

# Check top memory-consuming processes
ps aux --sort=-%mem | head -10

# Check top CPU-consuming processes
ps aux --sort=-%cpu | head -10
```

---

## 13. Archiving & Compression

```bash
# Create a .tar.gz archive
tar -czvf archive.tar.gz /path/to/directory

# Extract a .tar.gz archive
tar -xzvf archive.tar.gz

# Extract to a specific directory
tar -xzvf archive.tar.gz -C /target/dir/

# List contents of a .tar.gz archive without extracting
tar -tzvf archive.tar.gz

# Create a .zip archive
zip -r archive.zip /path/to/directory

# Extract a .zip archive
unzip archive.zip

# Extract .zip to specific directory
unzip archive.zip -d /target/dir/

# Compress a single file with gzip
gzip file.log

# Decompress a .gz file
gunzip file.log.gz
```

---

## 14. Git on the Server

```bash
# Install Git
sudo apt install -y git

# Configure global identity (first time)
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Clone a repository
git clone https://github.com/username/repo.git

# Clone a specific branch
git clone -b main https://github.com/username/repo.git

# Pull latest changes (for re-deploys)
cd /home/deploy/my-app
git pull origin main

# Check current branch and status
git status
git branch

# Stash local changes before pulling
git stash
git pull origin main
git stash pop

# Check recent commits
git log --oneline -10

# Roll back to a previous commit
git reset --hard <commit-hash>
```

---

## 15. Cron Jobs — Scheduled Tasks

```bash
# Open crontab editor for current user
crontab -e

# List current cron jobs
crontab -l

# Remove all cron jobs
crontab -r

# Edit system-wide cron jobs
sudo nano /etc/crontab
```

**Cron syntax:**

```
* * * * * /path/to/command
│ │ │ │ │
│ │ │ │ └─── Day of week (0=Sun, 6=Sat)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

**Common examples:**

```bash
# Run every minute
* * * * * /home/deploy/scripts/check.sh

# Run every day at midnight
0 0 * * * /home/deploy/scripts/backup.sh

# Run every Sunday at 2 AM
0 2 * * 0 /home/deploy/scripts/cleanup.sh

# Run every hour
0 * * * * /home/deploy/scripts/sync.sh

# Run every 5 minutes
*/5 * * * * /home/deploy/scripts/healthcheck.sh

# Redirect cron output to a log file
0 0 * * * /home/deploy/scripts/backup.sh >> /var/log/backup.log 2>&1
```

---

## 16. Useful Shortcuts & Tricks

```bash
# Repeat the last command with sudo
sudo !!

# Search command history
Ctrl + R   # then type to search

# Clear the terminal
clear
Ctrl + L

# Cancel a running command
Ctrl + C

# Pause a running command (resume with fg)
Ctrl + Z

# Run multiple commands sequentially
command1 && command2    # runs command2 only if command1 succeeds
command1 ; command2     # runs command2 regardless

# Run in background
command &

# Pipe output of one command into another
ps aux | grep node
cat access.log | grep "ERROR"

# Redirect output to a file (overwrite)
echo "hello" > file.txt

# Redirect output to a file (append)
echo "hello" >> file.txt

# Suppress output
command > /dev/null 2>&1

# Show a command's full path
which node
which python3
which java

# Show command history
history

# Run command #42 from history
!42

# Time how long a command takes
time npm install

# Watch a command's output every 2 seconds
watch -n 2 df -h
watch -n 1 "sudo ss -tlnp"

# Read a manual page
man nginx
man systemctl

# Quick help for a command
nginx --help
systemctl --help
```

---

## Quick Reference Card

| Task                   | Command                                                   |
| ---------------------- | --------------------------------------------------------- |
| Update server          | `sudo apt update && sudo apt upgrade -y`                  |
| Check disk space       | `df -h`                                                   |
| Check RAM              | `free -h`                                                 |
| Check running services | `sudo systemctl list-units --type=service --state=active` |
| View service logs      | `journalctl -u <service> -f`                              |
| Check open ports       | `sudo ss -tlnp`                                           |
| Firewall status        | `sudo ufw status numbered`                                |
| Find process on port   | `sudo lsof -i :<port>`                                    |
| Kill process by port   | `sudo kill -9 $(sudo lsof -t -i:<port>)`                  |
| Restart service        | `sudo systemctl restart <service>`                        |
| Check public IP        | `curl https://ipinfo.io/ip`                               |
| Sync files to server   | `rsync -avz ./app/ user@IP:/home/deploy/app/`             |
| Pull latest code       | `git pull origin main`                                    |
| Check CPU load         | `uptime`                                                  |
| Top processes          | `htop`                                                    |
| Search in logs         | `grep -i "error" /var/log/nginx/error.log`                |

---

_Last updated: June 2026_
