# Linux Fundamentals for Backend Engineers

A practical guide covering the essential Linux skills every backend engineer needs for development, deployment, and troubleshooting.

---

## 1. Command Line Essentials

### Navigation & File Operations
```bash
# Navigation
pwd                    # Print working directory
cd /var/log            # Change directory
cd ~                   # Go to home directory
cd -                   # Go to previous directory
ls -la                 # List all files with details

# File operations
touch app.log          # Create empty file
cp config.yml config.yml.backup    # Copy file
mv oldname.txt newname.txt         # Rename/move file
rm -rf node_modules/   # Remove directory recursively (be careful!)
mkdir -p projects/api/src          # Create nested directories
```

### Viewing & Searching Files
```bash
# Viewing files
cat server.log                    # Display entire file
head -n 50 error.log              # First 50 lines
tail -f /var/log/nginx/access.log # Follow log in real-time
less large-file.log               # Paginated view (press 'q' to quit)

# Searching with grep
grep "ERROR" app.log              # Find lines containing ERROR
grep -i "error" app.log           # Case-insensitive search
grep -r "TODO" src/               # Recursive search in directory
grep -E "ERROR|WARN" app.log      # Regex pattern (ERROR or WARN)
grep -v "DEBUG" app.log           # Exclude DEBUG lines

# Advanced with awk and sed
awk '{print $1}' access.log       # Print first column
awk '/ERROR/ {print $4, $5}' app.log    # Print columns 4,5 for ERROR lines
sed -i 's/localhost/127.0.0.1/g' config.yml   # Replace in-place
```

---

## 2. Process Management

### Viewing Processes
```bash
ps aux                          # All processes with details
ps aux | grep node              # Find Node.js processes
top                             # Interactive process viewer
htop                            # Enhanced top (if installed)

# Find processes by port
lsof -i :3000                   # What's using port 3000?
netstat -tulpn | grep :8080     # Alternative for port check
ss -tulpn | grep :8080          # Modern replacement for netstat
```

### Managing Processes
```bash
# Background jobs
node server.js &                # Run in background
jobs                            # List background jobs
fg %1                           # Bring job 1 to foreground
ctrl+z → bg                     # Suspend then background

# Killing processes
kill 1234                       # Send SIGTERM to PID 1234
kill -9 1234                    # Force kill (SIGKILL)
pkill -f "node server"          # Kill by pattern
killall node                    # Kill all node processes
```

---

## 3. File Permissions & Ownership

### Understanding Permissions
```
-rwxr-xr-x 1 user group 1234 Jan 1 12:00 script.sh
|  |  |  |
|  |  |  └── Others permissions
|  |  └───── Group permissions
|  └──────── User permissions
└─────────── File type (-=file, d=directory, l=link)

r = read (4)
w = write (2)
x = execute (1)
```

### Common Permission Commands
```bash
# Change permissions
chmod 755 deploy.sh             # rwxr-xr-x
chmod +x script.sh              # Make executable
chmod 644 config.yml            # rw-r--r-- (config files)
chmod 600 .env                  # rw------- (secrets)

# Change ownership
sudo chown user:group app/      # Change owner and group
sudo chown -R $USER:$USER .     # Change to current user recursively

# Special permissions
chmod +s binary                 # Setuid (run as owner)
chmod 1777 /tmp                 # Sticky bit (only owner can delete)
```

---

## 4. Networking Essentials

### Network Configuration
```bash
# IP and interface info
ip addr                         # Show IP addresses
ip link                         # Show network interfaces
hostname -I                     # Show all IP addresses

# Connectivity testing
ping google.com                 # Test connectivity
curl -I https://api.example.com # HTTP headers only
curl -v http://localhost:3000   # Verbose HTTP request
wget -O - http://localhost:8080/health  # Download to stdout
```

### Port & Connection Tools
```bash
# Check listening ports
ss -tlnp                        # TCP listening ports with processes
netstat -tlnp                   # Same (older systems)

# Connection testing
nc -zv localhost 3000           # Test if port is open (netcat)
telnet localhost 5432           # Test PostgreSQL port

# DNS lookup
nslookup api.example.com
dig api.example.com             # Detailed DNS info
```

### HTTP/API Testing with curl
```bash
# GET request
curl https://api.example.com/users

# POST with JSON
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com"}'

# With authentication
curl -H "Authorization: Bearer $TOKEN" \
  https://api.example.com/protected

# Save response
curl -o response.json https://api.example.com/data

# Follow redirects
curl -L https://bit.ly/xxx
```

---

## 5. Package Management

### Debian/Ubuntu (apt)
```bash
sudo apt update                 # Update package lists
sudo apt upgrade                # Upgrade installed packages
sudo apt install nginx postgresql redis-server
sudo apt remove nginx           # Remove package
sudo apt autoremove             # Clean up unused dependencies
apt search nodejs               # Search for package
apt show nginx                  # Package details
```

### RHEL/CentOS/Fedora (yum/dnf)
```bash
sudo yum update                 # Update packages
sudo yum install nginx
sudo yum remove nginx
sudo yum search nodejs

# Fedora uses dnf (next-gen)
sudo dnf install nginx
```

### Language-Specific Package Managers
```bash
# Node.js (npm/yarn/pnpm)
npm install                     # Install dependencies
npm run build                   # Run build script

# Python (pip)
pip install -r requirements.txt
pip install requests

# Go
go mod download
go build ./...
```

---

## 6. Systemd Service Management

### Essential Commands
```bash
# Service control
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx     # Graceful reload (config changes)

# Service status
sudo systemctl status nginx
sudo systemctl is-active nginx  # Check if running
sudo systemctl is-enabled nginx # Check if starts on boot

# Enable/disable on boot
sudo systemctl enable nginx
sudo systemctl disable nginx

# View logs
sudo journalctl -u nginx        # Service logs
sudo journalctl -u nginx -f     # Follow logs
sudo journalctl -u nginx --since "1 hour ago"
sudo journalctl -u nginx -n 100 # Last 100 lines
```

### Creating a Service
```bash
# /etc/systemd/system/myapp.service
[Unit]
Description=My Backend API
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload    # Reload systemd
sudo systemctl enable myapp
sudo systemctl start myapp
```

---

## 7. Shell Scripting Basics

### Variables & Conditionals
```bash
#!/bin/bash

# Variables
DB_HOST="localhost"
DB_PORT=5432
LOG_FILE="/var/log/app.log"

# Conditionals
if [ -f "$LOG_FILE" ]; then
    echo "Log file exists"
fi

if [ "$ENV" = "production" ]; then
    echo "Production mode"
else
    echo "Development mode"
fi

# Numeric comparison
if [ "$PORT" -eq 3000 ]; then
    echo "Default port"
fi
```

### Loops & Functions
```bash
#!/bin/bash

# For loop
for env in dev staging prod; do
    echo "Deploying to $env"
done

# While loop
while [ ! -f "/tmp/ready" ]; do
    echo "Waiting..."
    sleep 1
done

# Function
health_check() {
    local url=$1
    local status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
    if [ "$status" -eq 200 ]; then
        echo "OK"
        return 0
    else
        echo "FAILED: $status"
        return 1
    fi
}

health_check "http://localhost:3000/health"
```

### Useful Script Patterns
```bash
#!/bin/bash
set -euo pipefail    # Exit on error, undefined vars, pipe failures

# Logging with timestamps
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# Check if command exists
command -v docker >/dev/null 2>&1 || { echo "docker required"; exit 1; }

# Default values
PORT="${PORT:-3000}"
HOST="${HOST:-0.0.0.0}"
```

---

## 8. Log Management

### Viewing Logs
```bash
# System logs
sudo journalctl                   # All systemd logs
sudo journalctl -xe               # Recent logs with errors
sudo journalctl -u nginx -f       # Follow nginx logs

# Application logs
tail -f /var/log/myapp/app.log
grep "ERROR" /var/log/myapp/*.log | tail -20

# Log rotation status
logrotate -d /etc/logrotate.conf  # Debug mode (dry run)
```

### Log Rotation Configuration
```bash
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0644 appuser appgroup
    sharedscripts
    postrotate
        systemctl reload myapp
    endscript
}
```

---

## 9. Environment & Configuration

### Environment Variables
```bash
# View environment
env                             # All environment variables
printenv PATH                   # Specific variable
echo $HOME

# Set variables
export DATABASE_URL="postgresql://localhost/mydb"
export DEBUG=true

# Load from file
source .env                     # or: . .env
set -a && source .env && set +a # Export all vars from file

# Persistent environment
# Add to ~/.bashrc or ~/.bash_profile
export PATH="$HOME/.local/bin:$PATH"
```

### SSH Key Management
```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your@email.com"
ssh-keygen -t rsa -b 4096 -C "your@email.com"

# Copy public key to server
ssh-copy-id user@server.com

# SSH config (~/.ssh/config)
Host prod-api
    HostName 203.0.113.10
    User deploy
    IdentityFile ~/.ssh/prod_key
    Port 2222

# Usage
ssh prod-api
```

---

## 10. Docker Essentials

### Basic Commands
```bash
# Images
docker images                   # List images
docker pull nginx:latest
docker rmi image_id             # Remove image

# Containers
docker ps                       # Running containers
docker ps -a                    # All containers
docker run -d -p 3000:3000 --name api myapp:latest
docker stop api
docker rm api                   # Remove container
docker logs -f api              # Container logs

# Execute commands
docker exec -it api bash        # Shell in container
docker exec api ls -la /app     # Single command
```

### Docker Compose
```bash
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://db:5432/myapp
    depends_on:
      - db
  
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
docker-compose up -d            # Start in background
docker-compose down             # Stop and remove
docker-compose logs -f api      # Follow logs
docker-compose build            # Rebuild images
docker-compose exec api bash    # Shell in service
```

---

## 11. Useful One-Liners for Backend Engineers

```bash
# Find largest files/directories
du -h --max-depth=1 | sort -hr | head -10

# Count files by extension
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -nr

# Monitor HTTP requests in real-time
sudo tcpdump -i any port 80 -A | grep -E "GET|POST|PUT|DELETE"

# Generate random password
openssl rand -base64 32

# Test API endpoint with timing
curl -w "@curl-format.txt" -o /dev/null -s https://api.example.com/health

# Find and replace in all files
find . -type f -name "*.js" -exec sed -i 's/old/new/g' {} +

# Backup database
pg_dump -h localhost -U postgres mydb > backup_$(date +%Y%m%d).sql

# Check disk I/O
iostat -x 1

# Monitor network connections
watch -n 1 'ss -s'

# Kill all processes matching pattern
ps aux | grep "[n]ode" | awk '{print $2}' | xargs kill -9
```

---

## 12. Quick Reference Card

| Task | Command |
|------|---------|
| Find file | `find / -name "config.yml" 2>/dev/null` |
| Find in files | `grep -r "pattern" /path` |
| Disk usage | `df -h` |
| Directory size | `du -sh /var/log` |
| Memory usage | `free -h` |
| CPU usage | `top` or `htop` |
| Open files | `lsof -p <pid>` |
| Network connections | `ss -tulpn` |
| Download file | `curl -O URL` or `wget URL` |
| Extract tar | `tar -xzf file.tar.gz` |
| Create tar | `tar -czf backup.tar.gz folder/` |
| Edit file | `nano file` or `vim file` |
| File checksum | `sha256sum file` |
| Compare files | `diff file1 file2` |

---

## Resources for Further Learning

- **The Linux Command Line** by William Shotts (free PDF)
- **Linuxize** (linuxize.com) - Practical tutorials
- **Explainshell** (explainshell.com) - Decode command arguments
- **tldr-pages** (tldr.sh) - Simplified man pages

---

*Practice these commands in a safe environment. Many commands (especially with `sudo`, `rm -rf`, or `>` redirection) can be destructive.*
