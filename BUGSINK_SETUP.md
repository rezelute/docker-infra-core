# Bugsink Setup Guide

Lightweight Sentry-compatible error monitoring (~50MB vs GlitchTip's 510MB).

## Initial Deployment

### Set Environment Variables

Create a `.env` file in the project directory:

```bash
cat > .env << EOF
BUGSINK_ADMIN_EMAIL=admin@example.com
BUGSINK_ADMIN_PASSWORD=CHANGEME_SECURE_PASSWORD
EOF
```

**Security Note:** The `.env` file is gitignored. Keep your password secure.

### Start the Infrastructure Stack

```bash
docker compose up -d
```

### Check Bugsink Status

```bash
docker compose ps bugsink
docker compose logs -f bugsink
```

## First-Time Configuration

### 1. Access Bugsink

Navigate to: **https://bugsink.example.com**

### 2. Log In

- Use the email and password from your `.env` file
- The admin account is automatically created on first startup
- **No signup page available** - registration is disabled for security

### 3. Create Project

- Click "New Project"
- Name: e.g., "My New Project"
- Platform: Select "Node.js"

### 4. Get DSN

- Go to Project Settings
- Copy the DSN URL

Example DSN:

```
https://<key>@bugsink.example.com/1
```

### 5. Add to your Project

Add the DSN to your app's `.env` file:

```bash
BUGSINK_DSN=https://<key>@bugsink.example.com/1
```

## Integrating with Your App

### Install Sentry SDK (Sentry-compatible)

```bash
npm install @sentry/node
```

### Configure in your app

```typescript
// src/monitoring/sentry.ts
import * as Sentry from "@sentry/node"

Sentry.init({
   dsn: process.env.BUGSINK_DSN,
   environment: process.env.NODE_ENV,

   // Optional: Sample rate for performance monitoring
   tracesSampleRate: 0.1,

   // Sanitize sensitive data
   beforeSend(event) {
      // Remove sensitive headers
      if (event.request?.headers) {
         delete event.request.headers["authorization"]
         delete event.request.headers["cookie"]
         delete event.request.headers["x-delete-token"]
      }

      // Remove sensitive query params
      if (event.request?.query_string) {
         event.request.query_string = event.request.query_string
            .replace(/token=[^&]+/g, "token=[REDACTED]")
            .replace(/email=[^&]+/g, "email=[REDACTED]")
      }

      return event
   },
})

export default Sentry
```

### Add to your Express app

```typescript
// src/index.ts
import Sentry from "./monitoring/sentry"

// Initialize Sentry BEFORE other middleware
Sentry.setupExpressErrorHandler(app)

// ... rest of your app setup

// Add Sentry error handler AFTER all routes, BEFORE other error middleware
app.use(Sentry.Handlers.errorHandler())
```

### Test it

```typescript
// Trigger a test error
Sentry.captureException(new Error("Test error from My Application"))

// Or test with an endpoint
app.get("/debug-sentry", (req, res) => {
   throw new Error("Test Sentry error")
})
```

## Environment Variables

### Development (.env)

```bash
BUGSINK_DSN=https://<key>@bugsink.example.com/1
```

### Production (docker-compose.yml)

```yaml
environment:
   BUGSINK_DSN: "${BUGSINK_DSN}"
```

## Features

### What Bugsink Tracks:

- ✅ JavaScript/Node.js errors
- ✅ Unhandled promise rejections
- ✅ Stack traces
- ✅ Request context (URL, headers, user)
- ✅ Breadcrumbs (event trail)
- ✅ Environment info
- ✅ Release tracking

### Limitations vs Full Sentry:

- ⚠️ No performance monitoring (APM)
- ⚠️ No session replay
- ⚠️ Simpler UI (but still very usable)
- ⚠️ SQLite backend (fine for small teams)

## Maintenance

### View Database Size

```bash
docker compose exec bugsink du -h /data/db.sqlite3
```

### Backup Database

```bash
docker compose exec bugsink sqlite3 /data/db.sqlite3 ".backup /data/backup.db"
docker cp bugsink:/data/backup.db ./bugsink-backup-$(date +%Y%m%d).db
```

### Restore Database

```bash
docker cp bugsink-backup.db infra-bugsink:/data/restore.db
docker compose exec bugsink sqlite3 /data/db.sqlite3 ".restore /data/restore.db"
```

### View Logs

```bash
docker compose logs -f bugsink
```

### Update Bugsink

```bash
docker compose pull bugsink
docker compose up -d bugsink
```

## Resource Usage

Memory usage:

- **Idle:** ~40MB
- **Active:** ~60MB

Disk usage:

- **Fresh install:** ~20MB
- **With data:** Grows slowly (SQLite is efficient)

## Troubleshooting

### Can't access UI

```bash
# Check if service is running
docker compose ps bugsink

# Check logs
docker compose logs bugsink

# Restart service
docker compose restart bugsink
```

### Errors not appearing

1. Check DSN is correct in your app
2. Verify app can reach Bugsink (check network)
3. Look for errors in Bugsink logs
4. Test with `Sentry.captureException(new Error("Test"))`

### SQLite locked errors

SQLite handles concurrency well, but if you see lock errors:

```bash
# Restart Bugsink
docker compose restart bugsink
```

## Best Practices

1. **Always sanitize before sending:**
   - Remove tokens, passwords, API keys
   - Redact email addresses if needed
   - Strip sensitive query parameters

2. **Add context:**

   ```typescript
   Sentry.setUser({ id: user.id }) // Don't include email/name
   Sentry.setTag("feature", "account-deletion")
   ```

3. **Use breadcrumbs:**

   ```typescript
   Sentry.addBreadcrumb({
      message: "User clicked delete account",
      level: "info",
   })
   ```

4. **Set environment:**
   ```typescript
   environment: process.env.NODE_ENV // "production", "development"
   ```

## Resources

- [Bugsink Documentation](https://bugsink.com/docs/)
- [Sentry SDK for Node.js](https://docs.sentry.io/platforms/node/)
- [Error Tracking Best Practices](https://docs.sentry.io/product/issues/)
