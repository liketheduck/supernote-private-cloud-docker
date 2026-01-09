# Supernote Private Cloud - Docker Setup

A secure, self-hosted Supernote Private Cloud deployment using Docker Compose with HTTPS, persistent storage, and email support.

## Architecture

This setup deploys a 4-service stack:

- **MariaDB 10.6.24**: Database server for Supernote data
- **Redis 7.4.7**: Cache layer with password authentication
- **Notelib 6.9.3**: Supernote library service for document conversion (note viewing)
- **Supernote Service 25.12.17**: Main application with HTTPS, REST API, and WebSocket support

All services communicate through a private Docker bridge network (`supernote-net`). The MariaDB and Redis services are not exposed to host ports by default—only the Supernote service is accessible via HTTPS. MariaDB can optionally be exposed for integrations like [supernote-apple-reminders-sync](https://github.com/liketheduck/supernote-apple-reminders-sync).

## Prerequisites

### Docker

**For macOS:** Install [Docker Desktop](https://www.docker.com/products/docker-desktop/) or [OrbStack](https://orbstack.dev/).

- **Apple Silicon (M1-M4):** OrbStack is recommended. The Supernote images are x86-based and require emulation on ARM Macs. OrbStack has a lighter footprint and better x86 emulation performance than Docker Desktop. It's a drop-in replacement.
- **Intel Macs:** Either option works well.

**For Linux:** Install [Docker Engine](https://docs.docker.com/engine/install/) (includes the `docker compose` command).

## Quick Start

### 1. Clone and Configure

```bash
# Clone the repository
git clone https://github.com/liketheduck/supernote-private-cloud-docker.git
cd supernote-private-cloud-docker

# Copy example files
cp docker-compose.example.yml docker-compose.yml
cp .env.example .env

# Edit .env with your settings (generate secure passwords)
nano .env
```

### 2. Generate Secure Passwords

```bash
# Generate random passwords for .env
openssl rand -base64 24  # Use for MYSQL_ROOT_PASSWORD
openssl rand -base64 24  # Use for MYSQL_PASSWORD
openssl rand -base64 24  # Use for REDIS_PASSWORD
```

### 3. Update Volume Paths

Edit `docker-compose.yml` and update the volume paths to match your system. The example uses relative paths (`./data/`) which work for most setups.

### 4. Generate SSL Certificate

```bash
# Create cert directory
mkdir -p cert

# Generate self-signed certificate (valid 10 years)
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout cert/server.key \
  -out cert/server.crt \
  -subj "/CN=your-hostname.local" \
  -addext "subjectAltName=DNS:your-hostname.local,DNS:localhost,IP:127.0.0.1"
```

### 5. Start Services

```bash
docker compose up -d
```

Check status:
```bash
docker compose ps
```

## Storage Structure

```
./
├── data/
│   ├── mariadb/          # Database files
│   ├── redis/            # Cache persistence
│   ├── supernote/        # User documents
│   ├── recycle/          # Deleted files
│   └── convert/          # Document conversions
├── cert/                 # HTTPS certificates
└── logs/
    ├── cloud/            # Application cloud logs
    ├── app/              # Application service logs
    └── web/              # Nginx web server logs
```

## Network Configuration

| Port | Type | Purpose |
|------|------|---------|
| **19443** | HTTPS | Primary access (secure) |
| **19072** | HTTP | Fallback/redirect to HTTPS |
| **18072** | WebSocket | Document auto-sync between devices |
| **3306** | TCP | MariaDB (optional, for integrations) |

## Initial Setup

### 1. Access the Web Interface

```
https://your-hostname.local:19443
```

Or via IP address:
```
https://YOUR_IP_ADDRESS:19443
```

### 2. Device Connection

On your Supernote device:
- **Server**: Your hostname or IP address
- **Port**: `19443`
- **Enable "Allow insecure connections"** (for self-signed certificate)

### 3. Email Configuration (Optional)

Email is required for user registration. Configure via the web interface:

1. Click **"Email Settings"** button
2. Set:
   - **SMTP Server**: `smtp.gmail.com`
   - **Port**: `465`
   - **Email**: Your email address
   - **Password**: App-specific password (not your regular password)
   - **Encryption Type**: `SSL`
3. Click **Save** and **Test**

**For Gmail users:**
1. Enable 2-factor authentication on your Google account
2. Go to [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
3. Generate an app password
4. Use the 16-character password in email settings

Alternatively, copy and configure `supernote-email-init.sql.example` to pre-populate email settings.

### 4. User Registration

1. Click **Register** in the web interface
2. Enter your email (will receive verification code)
3. Enter the code from the email
4. Complete registration

## Configuration Files

### `.env` - Environment Variables

```
MYSQL_ROOT_PASSWORD=     # Root database password
MYSQL_USER=supernote     # Database user
MYSQL_PASSWORD=          # Database user password
REDIS_PASSWORD=          # Redis password
DOMAIN_NAME=             # Hostname/domain for HTTPS
SSL_CERT_NAME=server.crt # Certificate filename
SSL_KEY_NAME=server.key  # Key filename
```

### `docker-compose.yml` - Service Configuration

Services are configured with:
- **Health checks**: Automatic restart on failure
- **Dependencies**: Services wait for required services to be healthy
- **Restart policy**: `unless-stopped` (survives host reboot)
- **Networks**: Private `supernote-net` bridge (services isolated from other Docker containers)
- **Notelib alias**: The notelib container must have the network alias `notelib` (the supernote-service has this hostname hardcoded)

## Commands

### Stop Services
```bash
docker compose down
```

### View Logs
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f supernote-service
```

### Database Backup
```bash
# Backup
docker compose exec supernote-mariadb mysqldump -uroot -p$MYSQL_ROOT_PASSWORD supernotedb > backup.sql

# Restore
docker compose exec -T supernote-mariadb mysql -uroot -p$MYSQL_ROOT_PASSWORD supernotedb < backup.sql
```

## Troubleshooting

### Services won't start

```bash
# Check container logs
docker compose logs supernote-service

# Verify network is created
docker network ls | grep supernote-net

# Check disk space
df -h
```

### HTTPS certificate errors

This is normal with a self-signed certificate. Your browser will warn you—accept the certificate exception to continue.

On mobile devices, you may need to install the certificate in your device's certificate store.

### Email not sending

Check email configuration via **Email Settings** > **Test**. Common issues:

- **Wrong app password**: Use the 16-character password from your email provider, not your regular password
- **2FA not enabled**: Gmail app passwords require 2-factor authentication
- **Port blocked**: Some networks block port 465. Try port 587 with TLS instead of SSL

### Database connection errors

```bash
# Verify MariaDB is healthy
docker compose logs supernote-mariadb | grep -i error

# Check data directory permissions
ls -la ./data/mariadb/
```

### Note conversion not working (viewing notes in browser)

The notelib service converts `.note` files to PNG for browser viewing. After conversion, notelib must upload the images back to supernote-service via HTTPS.

**With self-signed certificates**, notelib cannot trust the certificate and uploads fail:
```
curl_easy_perform() failed: SSL peer certificate or SSH remote key was not OK
```

**Solution - Create a CA bundle with your self-signed cert:**

The `docker-compose.example.yml` already includes the volume mount for this. You just need to create the CA bundle file:

```bash
# Start containers first (if not already running)
docker compose up -d

# Extract the default CA bundle from notelib container and append your cert
docker exec supernote-notelib cat /etc/ssl/certs/ca-certificates.crt > cert/ca-bundle.crt
cat cert/server.crt >> cert/ca-bundle.crt

# Restart to apply
docker compose down && docker compose up -d
```

**Check logs** to confirm the issue or verify the fix:
```bash
docker logs supernote-notelib --tail 50 | grep -i "SSL\|certificate\|upload"
```

## Maintenance

### Checking for Updates

Supernote doesn't publish a compatibility matrix, so there's no official signal when to update. The safest approach:

**1. Check Supernote's official docker-compose for version changes:**
```bash
curl -s https://supernote-private-cloud.supernote.com/docker-deploy/docker-compose.yml
```

When Supernote bumps MariaDB or Redis versions in their official file, it's safe to update yours.

**2. Check for new Supernote image versions on Docker Hub:**
- [supernote/supernote-service](https://hub.docker.com/r/supernote/supernote-service/tags)
- [supernote/notelib](https://hub.docker.com/r/supernote/notelib/tags)

**3. Conservative approach:** MariaDB 10.6.x and Redis 7.x are LTS/stable branches. Only update these when Supernote's official compose changes, or if you need security patches.

**4. Community resources:**
- [Supernote subreddit](https://reddit.com/r/supernote) — users often report issues with new versions
- [Supernote Support](https://support.supernote.com/) — official announcements

### Applying Updates

1. Update image versions in `docker-compose.yml`
2. Pull and restart: `docker compose up -d`
3. Docker will automatically pull new images

### Rotate SSL Certificate

1. Generate new certificate files
2. Copy to `cert/` directory
3. Restart: `docker compose restart supernote-service`

## Security Considerations

- **Self-signed HTTPS**: Protects data in transit on local network
- **Database password**: Stored in `.env` (not committed to git)
- **Redis password**: Password-protected; not exposed to network
- **Private network**: Services isolated from other containers
- **No public exposure**: Only accessible on local network by default

## Files Overview

```
./
├── docker-compose.yml          # Service definitions (from example)
├── docker-compose.example.yml  # Example configuration
├── .env                        # Environment variables (from example)
├── .env.example                # Example environment
├── supernotedb.sql             # Database schema
├── supernote-email-init.sql.example  # Email config template
├── cert/                       # SSL certificates
├── data/                       # Persistent data
├── logs/                       # Application logs
└── README.md                   # This file
```

## License

This Docker configuration is provided as-is for self-hosting Supernote Private Cloud. Supernote and related services are trademarks of Ratta Software.
