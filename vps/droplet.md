# DigitalOcean Droplet Setup: Node.js & Flask API Deployment

## Prerequisites

- DigitalOcean account
- A domain name (optional but recommended)
- SSH key pair generated locally

---

## 1. Create a Droplet

1. Log in to [DigitalOcean](https://cloud.digitalocean.com)
2. Click **Create → Droplets**
3. Choose:
   - **Image:** Ubuntu 22.04 LTS
   - **Plan:** Basic → Regular → $6/mo (1 GB RAM / 1 vCPU) minimum
   - **Region:** Closest to your users
   - **Authentication:** SSH Key (recommended over password)
4. Click **Create Droplet**

---

## 2. Initial Server Setup

```bash
# Connect to your droplet
ssh root@YOUR_DROPLET_IP

# Update packages
apt update && apt upgrade -y

# Create a non-root user
adduser deploy
usermod -aG sudo deploy

# Copy SSH keys to new user
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy

# Switch to new user
su - deploy
```

---

## 3. Install Node.js

```bash
# Install Node.js via NodeSource (v20 LTS)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node -v   # e.g., v20.x.x
npm -v

# Install PM2 (process manager for Node.js)
sudo npm install -g pm2
```

---

## 4. Install Python & Flask

```bash
# Install Python and pip
sudo apt install -y python3 python3-pip python3-venv

# Verify
python3 --version
pip3 --version
```

---

## 5. Deploy Node.js API

```bash
# Clone your project (or upload via scp/rsync)
git clone https://github.com/yourusername/your-nodejs-api.git
cd your-nodejs-api

# Install dependencies
npm install

# Start with PM2
pm2 start index.js --name "nodejs-api"

# Save PM2 process list and enable on reboot
pm2 save
pm2 startup
# Run the command PM2 outputs to enable autostart
```

**Sample Node.js app (`index.js`) to test:**

```js
const express = require("express");
const app = express();

app.get("/", (req, res) => res.json({ status: "Node.js API running" }));

app.listen(3000, () => console.log("Listening on port 3000"));
```

---

## 6. Deploy Flask API

```bash
# Clone your Flask project
git clone https://github.com/yourusername/your-flask-api.git
cd your-flask-api

# Create and activate virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
pip install gunicorn  # Production WSGI server

# Test Flask app
gunicorn --bind 0.0.0.0:5000 app:app
```

**Sample Flask app (`app.py`) to test:**

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def index():
    return jsonify(status='Flask API running')

if __name__ == '__main__':
    app.run()
```

**Run Flask as a systemd service:**

```bash
sudo nano /etc/systemd/system/flask-api.service
```

Paste the following (adjust paths):

```ini
[Unit]
Description=Flask API
After=network.target

[Service]
User=deploy
WorkingDirectory=/home/deploy/your-flask-api
Environment="PATH=/home/deploy/your-flask-api/venv/bin"
ExecStart=/home/deploy/your-flask-api/venv/bin/gunicorn --workers 3 --bind 0.0.0.0:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable flask-api
sudo systemctl start flask-api
sudo systemctl status flask-api
```

---

## 7. Install & Configure Nginx (Reverse Proxy)

```bash
sudo apt install -y nginx

# Open firewall for Nginx
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable
```

Create an Nginx config file:

```bash
sudo nano /etc/nginx/sites-available/apis
```

Paste the following:

```nginx
# Node.js API — available at /api/node/
server {
    listen 80;
    server_name your-domain.com;  # or your Droplet IP

    # Node.js API
    location /api/node/ {
        proxy_pass http://localhost:3000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Flask API
    location /api/flask/ {
        proxy_pass http://localhost:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/apis /etc/nginx/sites-enabled/

# Test and reload Nginx
sudo nginx -t
sudo systemctl reload nginx
```

---

## 8. Secure with SSL (HTTPS) via Let's Encrypt

```bash
sudo apt install -y certbot python3-certbot-nginx

# Obtain certificate (replace with your domain)
sudo certbot --nginx -d your-domain.com

# Auto-renewal is set up automatically; verify with:
sudo certbot renew --dry-run
```

---

## 9. Configure Firewall (UFW)

```bash
sudo ufw status

# Allow required ports
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'

# Block direct access to app ports from outside
sudo ufw deny 3000
sudo ufw deny 5000

sudo ufw reload
```

---

## 10. Verify Everything Works

```bash
# Check Node.js API via PM2
pm2 list
pm2 logs nodejs-api

# Check Flask API via systemd
sudo systemctl status flask-api
journalctl -u flask-api -f

# Test endpoints
curl http://your-domain.com/api/node/
curl http://your-domain.com/api/flask/
```

---

## 11. Install Docker

```bash
# Remove any old Docker versions
sudo apt remove -y docker docker-engine docker.io containerd runc

# Install dependencies
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Allow current user to run Docker without sudo
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

---

## 12. Run PostgreSQL with Docker

Pull the official PostgreSQL image and start a container with a custom username, password, and database name:

```bash
docker run -d \
  --name postgres-db \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  --restart unless-stopped \
  postgres:16
```

**What each flag does:**

| Flag                       | Purpose                                  |
| -------------------------- | ---------------------------------------- |
| `-d`                       | Run container in background (detached)   |
| `--name postgres-db`       | Name the container for easy reference    |
| `POSTGRES_USER`            | PostgreSQL username                      |
| `POSTGRES_PASSWORD`        | PostgreSQL password                      |
| `POSTGRES_DB`              | Default database to create on startup    |
| `-p 5432:5432`             | Map host port 5432 → container port 5432 |
| `--restart unless-stopped` | Auto-restart on reboot or crash          |

**Verify the container is running:**

```bash
docker ps
docker logs postgres-db
```

**Connect to PostgreSQL inside the container:**

```bash
docker exec -it postgres-db psql -U myuser -d mydb
```

**Connection string for your Node.js or Flask app:**

```
postgresql://myuser:mysecretpassword@localhost:5432/mydb
```

**Persist data with a volume (recommended for production):**

```bash
docker run -d \
  --name postgres-db \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  --restart unless-stopped \
  postgres:16
```

> ⚠️ Without a volume, all data is lost if the container is removed.

---

## 13. Allow PostgreSQL Access from Your Local Machine Only

By default, port 5432 should **never** be open to the public internet. Use UFW to allow access only from your specific local machine IP.

### Step 1 — Find your local machine's public IP

Run this on your **local computer** (not the server):

```bash
# macOS / Linux
curl https://ipinfo.io/ip

# Windows (PowerShell)
(Invoke-WebRequest -Uri "https://ipinfo.io/ip").Content
```

### Step 2 — Allow only your IP on the Droplet

SSH into your Droplet and run:

```bash
# Replace 203.0.113.45 with your actual local machine IP
YOUR_LOCAL_IP=203.0.113.45

# Allow PostgreSQL port only from your IP
sudo ufw allow from $YOUR_LOCAL_IP to any port 5432 proto tcp

# Deny access from everyone else
sudo ufw deny 5432

# Reload firewall
sudo ufw reload

# Confirm the rules
sudo ufw status numbered
```

You should see output like:

```
[ 1] 5432/tcp                   ALLOW IN    203.0.113.45
[ 2] 5432                       DENY IN     Anywhere
```

### Step 3 — Connect from your local machine

You can now connect directly from your laptop using any PostgreSQL client:

```bash
# psql CLI
psql -h YOUR_DROPLET_IP -p 5432 -U myuser -d mydb

# Or using a connection string
psql "postgresql://myuser:mysecretpassword@YOUR_DROPLET_IP:5432/mydb"
```

GUI tools like **TablePlus**, **DBeaver**, or **pgAdmin** will also work with the same host/port/credentials.

### Dynamic IP? Use SSH Tunnel Instead

If your local IP changes frequently (home ISP), use an SSH tunnel — it's more secure and doesn't require opening port 5432 at all:

```bash
# On your local machine — forward local port 5433 to Droplet's 5432
ssh -L 5433:localhost:5432 deploy@YOUR_DROPLET_IP -N

# Then connect to localhost:5433 (tunneled securely over SSH)
psql -h localhost -p 5433 -U myuser -d mydb
```

> Keep the tunnel open in a terminal tab while you need the connection.

---

## 14. Deploy Spring Boot API

### Step 1 — Install Java (JDK 21 LTS)

```bash
sudo apt install -y openjdk-21-jdk

# Verify
java -version
# openjdk version "21.x.x" ...
```

### Step 2 — Build the JAR on Your Local Machine

In your Spring Boot project directory on your **local machine**:

```bash
# With Maven
./mvnw clean package -DskipTests

# With Gradle
./gradlew bootJar
```

The built JAR will be at:

- Maven: `target/your-app-0.0.1-SNAPSHOT.jar`
- Gradle: `build/libs/your-app-0.0.1-SNAPSHOT.jar`

### Step 3 — Upload the JAR to Your Droplet

```bash
# From your local machine
scp target/your-app-0.0.1-SNAPSHOT.jar deploy@YOUR_DROPLET_IP:/home/deploy/spring-api/app.jar
```

Or clone the repo directly on the server and build there:

```bash
# On the Droplet — install Maven/Gradle and build
sudo apt install -y maven
git clone https://github.com/yourusername/your-spring-api.git
cd your-spring-api
mvn clean package -DskipTests
cp target/*.jar /home/deploy/spring-api/app.jar
```

### Step 4 — Configure application.properties

Create an environment-specific config on the server to avoid hardcoding secrets:

```bash
mkdir -p /home/deploy/spring-api
nano /home/deploy/spring-api/application.properties
```

Paste and adjust:

```properties
server.port=8080

# PostgreSQL (matches Docker container from Section 12)
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=myuser
spring.datasource.password=mysecretpassword
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

### Step 5 — Run as a systemd Service

```bash
sudo nano /etc/systemd/system/spring-api.service
```

Paste the following (adjust paths and JAR name):

```ini
[Unit]
Description=Spring Boot API
After=network.target

[Service]
User=deploy
WorkingDirectory=/home/deploy/spring-api
ExecStart=/usr/bin/java -jar /home/deploy/spring-api/app.jar \
  --spring.config.location=file:/home/deploy/spring-api/application.properties
SuccessExitStatus=143
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable spring-api
sudo systemctl start spring-api

# Check status
sudo systemctl status spring-api

# Tail live logs
journalctl -u spring-api -f
```

### Step 6 — Add to Nginx Reverse Proxy

Open your existing Nginx config:

```bash
sudo nano /etc/nginx/sites-available/apis
```

Add the Spring Boot location block inside your `server { }` block:

```nginx
# Spring Boot API
location /api/spring/ {
    proxy_pass http://localhost:8080/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

```bash
# Test and reload Nginx
sudo nginx -t
sudo systemctl reload nginx
```

Spring Boot API is now accessible at `https://your-domain.com/api/spring/`.

### Step 7 — Block Direct Port Access

```bash
sudo ufw deny 8080
sudo ufw reload
```

### Updating the App (Re-deploy)

```bash
# Upload new JAR from local machine
scp target/your-app-*.jar deploy@YOUR_DROPLET_IP:/home/deploy/spring-api/app.jar

# Restart the service on the Droplet
sudo systemctl restart spring-api
journalctl -u spring-api -f
```

### Optional: Run with Docker Instead

If you prefer running Spring Boot in a Docker container alongside PostgreSQL:

**`Dockerfile`** (in your project root):

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**`docker-compose.yml`:**

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16
    container_name: postgres-db
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mysecretpassword
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  spring-api:
    build: .
    container_name: spring-api
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/mydb
      SPRING_DATASOURCE_USERNAME: myuser
      SPRING_DATASOURCE_PASSWORD: mysecretpassword
    depends_on:
      - postgres
    restart: unless-stopped

volumes:
  pgdata:
```

```bash
# Build and start both containers
docker compose up -d --build

# View logs
docker compose logs -f spring-api
```

> Using `docker-compose`, Spring Boot connects to Postgres via the service name `postgres` (Docker's internal DNS), not `localhost`.

---

## Port Reference

| Service         | Port | Access             |
| --------------- | ---- | ------------------ |
| Node.js API     | 3000 | Internal only      |
| Flask API       | 5000 | Internal only      |
| PostgreSQL      | 5432 | Your local IP only |
| Spring Boot API | 8080 | Internal only      |
| Nginx HTTP      | 80   | Public             |
| Nginx HTTPS     | 443  | Public (SSL)       |
| SSH             | 22   | Public (key)       |

---

## Quick Troubleshooting

| Issue                     | Command                             |
| ------------------------- | ----------------------------------- |
| Nginx not responding      | `sudo systemctl restart nginx`      |
| Flask API not starting    | `journalctl -u flask-api -n 50`     |
| Node.js app crashed       | `pm2 logs nodejs-api --lines 50`    |
| Spring Boot not starting  | `journalctl -u spring-api -n 50`    |
| Spring Boot restart       | `sudo systemctl restart spring-api` |
| Check port conflicts      | `sudo ss -tlnp`                     |
| Certificate renewal       | `sudo certbot renew`                |
| PostgreSQL container down | `docker start postgres-db`          |
| View PostgreSQL logs      | `docker logs postgres-db`           |
| List all containers       | `docker ps -a`                      |
| Docker Compose restart    | `docker compose restart spring-api` |
| Check UFW rules           | `sudo ufw status numbered`          |
| Remove UFW rule by number | `sudo ufw delete <number>`          |

---

_Last updated: June 2026_
