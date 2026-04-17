# Azure VM Deployment Guide for CISO Assistant Enterprise

## Prerequisites

- An Azure VM (Ubuntu 22.04+ recommended) with:
  - [Docker Engine](https://docs.docker.com/engine/install/ubuntu/) installed
  - [Docker Compose V2](https://docs.docker.com/compose/install/) installed (ships with Docker Engine)
- SSH access to the VM
- The `docker-compose-enterprise.yml` file (gitignored — must be transferred separately)

## 1. Prepare the VM

SSH into the VM and clone the repository:

```bash
ssh <user>@<vm-ip>
git clone https://github.com/intuitem/ciso-assistant-community.git
cd ciso-assistant-community
```

Copy the enterprise compose file to the VM (run from your local machine):

```bash
scp docker-compose-enterprise.yml <user>@<vm-ip>:~/ciso-assistant-community/
```

## 2. Configure Environment Variables

Edit `docker-compose-enterprise.yml` on the VM and update the following values.

### Required changes

| Variable | Service(s) | Change to |
|----------|-----------|-----------|
| `CISO_ASSISTANT_URL` | backend, huey, caddy | `https://<vm-hostname-or-ip>:8443` |
| `ALLOWED_HOSTS` | backend, huey | `backend,localhost,<vm-hostname-or-ip>` |
| `PUBLIC_BACKEND_API_EXPOSED_URL` | frontend | `https://<vm-hostname-or-ip>:8443/api` |
| `CISO_ASSISTANT_SUPERUSER_EMAIL` | backend | Your admin email address |
| `DJANGO_DEBUG` | backend | `False` (must be False in production) |

### Enterprise license

| Variable | Service(s) | Description |
|----------|-----------|-------------|
| `LICENSE_SEATS` | backend, huey | Number of licensed seats |
| `LICENSE_EXPIRATION` | backend, huey | License expiration date (`YYYY-MM-DD`) |

## 3. Deploy

```bash
docker compose -f docker-compose-enterprise.yml up -d --build
```

The backend health check (`/api/health/`) will retry for up to 5 minutes. Monitor startup:

```bash
docker compose -f docker-compose-enterprise.yml logs -f
```

Once all services are healthy, access CISO Assistant at:

```
https://<vm-ip>:8443
```

On first launch the superuser account is created automatically using `CISO_ASSISTANT_SUPERUSER_EMAIL`. Check the backend logs for the generated password:

```bash
docker compose -f docker-compose-enterprise.yml logs backend | grep -i password
```

## 4. Azure Network Security Group (NSG)

Ensure port **8443** is open for inbound TCP traffic in the VM's Network Security Group:

1. Azure Portal > VM > **Networking** > **Inbound port rules**
2. Add rule: **Destination port** `8443`, **Protocol** TCP, **Action** Allow
3. Restrict the **Source** to your IP range or VPN CIDR for security

## 5. TLS Configuration

### Default (self-signed)

Caddy generates a self-signed certificate via `tls internal`. Browsers will show a security warning.

### Production (Let's Encrypt)

If the VM has a public DNS name (e.g., `ciso.example.com`):

1. Point the DNS A record to the VM's public IP
2. Update `CISO_ASSISTANT_URL` to `https://ciso.example.com` (port 443)
3. Change the Caddy port mapping from `8443:8443` to `443:443` (and add `80:80` for the ACME HTTP challenge)
4. Remove `tls internal` from the Caddy command block so Caddy uses automatic HTTPS

### Custom certificate

Replace the Caddy `tls internal` line with:

```
tls /path/to/cert.pem /path/to/key.pem
```

And mount the certificate files into the Caddy container via an additional volume.

## 6. Persistence and Backups

The SQLite database is stored in the `./db/` volume mount on the VM. Back it up regularly:

```bash
# Simple cron backup (daily at 2 AM)
echo '0 2 * * * cp ~/ciso-assistant-community/db/ciso-assistant.sqlite3 ~/backups/ciso-assistant-$(date +\%F).sqlite3' | crontab -

# Or sync to Azure Blob Storage
azcopy copy ./db/ciso-assistant.sqlite3 "https://<storage-account>.blob.core.windows.net/<container>/backups/ciso-assistant-$(date +%F).sqlite3"
```

For PostgreSQL in production, use `pg_dump` instead.

## 7. Updating

```bash
cd ~/ciso-assistant-community

# Pull latest code
git pull

# Re-copy docker-compose-enterprise.yml if it changed locally
# (it's gitignored, so git pull won't update it)

# Rebuild and restart
docker compose -f docker-compose-enterprise.yml up -d --build
```

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `502 Bad Gateway` | Backend is still starting. Wait for the health check to pass: `docker compose -f docker-compose-enterprise.yml ps` |
| Can't reach port 8443 | Check Azure NSG rules and VM firewall (`sudo ufw status`) |
| "CSRF verification failed" | Ensure `CISO_ASSISTANT_URL` and `ALLOWED_HOSTS` match the URL you're using in the browser |
| Permission denied on `./db/` | Run `sudo chown -R 1000:1000 ./db/` to fix volume ownership |
