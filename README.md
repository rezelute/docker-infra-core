# Infra-Docker: Shared Infrastructure Services

This directory contains shared infrastructure services for all applications on this VPS. It provides a reverse proxy (Caddy), container management (Dockhand), and error monitoring (Bugsink).

## Services

- **Caddy**: Reverse proxy with automatic SSL/TLS (Let's Encrypt)
- **Dockhand**: Container management UI
- **Bugsink**: Lightweight error monitoring (Sentry-compatible)

## Structure

```
infra-docker/
├── compose.yml           # Infrastructure services
├── Caddyfile             # Caddy routing configuration
└── secrets/
  └── bugsink_secret_key.txt
```

## Usage

### Initial Setup

1. **Start the infrastructure** (network created automatically):

   ```bash
   cd /path/to/docker-infra-core
   docker compose up -d
   ```

2. **Verify services are running**:
   ```bash
   docker compose ps
   ```

### Adding New Applications

When deploying new apps, ensure they:

1. Connect to the `caddy_network` (set as external)
2. Use container names that match Caddyfile routing
3. Only expose internal ports (no public ports needed)

Example in app's compose.yml:

```yaml
services:
   frontend:
      expose:
         - "80"
      networks:
         - caddy_network

networks:
   caddy_network:
      external: true
      name: caddy_network
```

Then add routes to the Caddyfile:

```
app2.yourdomain.com {
    reverse_proxy app2-frontend-1:80
}
```

### Updating Caddyfile

After editing Caddyfile:

```bash
   docker compose restart caddy
```

### Accessing Services

- **Dockhand UI**: Accessible via Tailscale at your configured port
- **Bugsink**: https://bugsink-ingest.example.com
- **Apps**: Configured via Caddyfile (e.g., https://your-app.example.com)

### Bugsink Setup

1. **First-time access**: Navigate to https://bugsink-ingest.example.com
2. **Login**: Use admin credentials set in environment variables
3. **Create project**: Add a project (e.g., "My Application")
4. **Get DSN**: Copy the DSN URL for your app configuration

Example DSN format:

```
https://<key>@bugsink-ingest.example.com/<project-id>
```

## Maintenance

### View logs

```bash
docker compose logs -f caddy
docker compose logs -f dockhand
docker compose logs -f bugsink
```

### Update images

```bash
docker compose pull
docker compose up -d
```

### Backup Dockhand data

```bash
docker run --rm -v myinfra_dockhand_data:/app/data -v $(pwd):/backup alpine tar czf /backup/dockhand-backup.tar.gz -C /app/data .
```

## Security Notes

- Bugsink secret key is stored in `secrets/bugsink_secret_key.txt` and managed as a Docker secret
- Caddy automatically manages SSL certificates via Let's Encrypt
- Docker socket is mounted for Dockhand (required for container management)
- Some services like: Dockhand, Bugsink etc should ideally be exposed via Tailscale only (not public)
- All services restart automatically unless manually stopped
