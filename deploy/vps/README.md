# VPS Deployment

This directory contains a production-oriented VPS deployment template for `nardinmarcus/jmapi`.

## Recommended Topology

```text
Internet
  -> Nginx or Caddy on 443
  -> 127.0.0.1:5100
  -> jmapi container
  -> Jimeng/Dreamina upstream APIs
```

The API container is intentionally bound to `127.0.0.1:5100` in `docker-compose.yml`. Do not expose port `5100` directly to the public internet unless the VPS is protected by a firewall or private network.

## Prepare the VPS

```bash
mkdir -p /opt/jmapi
cd /opt/jmapi
```

Copy this directory's files into `/opt/jmapi`:

```text
/opt/jmapi/
  docker-compose.yml
  configs/
    prod/
      service.yml
      system.yml
  logs/
  tmp/
```

Create the runtime directories:

```bash
mkdir -p logs tmp
```

## Publish the Fork Image

The compose file uses:

```text
ghcr.io/nardinmarcus/jmapi:latest
```

The repository already includes a GitHub Actions workflow for GHCR publishing. Push a tag such as `v1.0.0`, or run the `Docker Publish` workflow manually after a semver tag exists.

If the fork image is not published yet, use one of these temporary alternatives:

```yaml
image: ghcr.io/iptag/jimeng-api:latest
```

Or clone your fork on the VPS and build from source:

```bash
git clone https://github.com/nardinmarcus/jmapi.git /opt/jmapi-src
cd /opt/jmapi-src
docker build -t jmapi:local .
```

Then change `/opt/jmapi/docker-compose.yml` to:

```yaml
image: jmapi:local
```

## Start the Service

```bash
docker compose up -d
docker compose ps
curl -sS http://127.0.0.1:5100/ping
```

If the VPS still uses Compose v1, replace `docker compose` with `docker-compose`.

Expected health response:

```text
pong
```

## Configure Nginx

Copy `nginx.conf` into your Nginx site configuration and replace every `api.example.com` with your real domain.

Important settings:

- `client_max_body_size 100m` matches the app upload limit.
- `proxy_read_timeout 3700s` allows long video generation polling.
- The `/ping` endpoint stays lightweight for uptime checks.

After editing:

```bash
nginx -t
systemctl reload nginx
```

## Verify Public Access

```bash
curl -sS https://api.example.com/ping
```

Check token credit access:

```bash
curl -X POST https://api.example.com/token/points \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_SESSION_ID"
```

## Token-Level Proxy

The service supports binding a proxy to a single Jimeng/Dreamina session token. This is useful when a specific account or region needs a different outbound IP, without forcing the whole container through a global proxy.

Format:

```text
<proxy-url>@<region-prefixed-session-id>
```

Examples:

```text
socks5://127.0.0.1:1080@us-YOUR_SESSION_ID
http://user:pass@proxy.example.com:8080@hk-YOUR_SESSION_ID
```

Multiple tokens can still be passed as a comma-separated list:

```bash
curl -X POST https://api.example.com/token/points \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer socks5://127.0.0.1:1080@us-TOKEN1,hk-TOKEN2,TOKEN3"
```

Only use trusted proxy endpoints. The proxy URL and session token are both sensitive credentials.

## Operations

View logs:

```bash
docker logs -f jmapi
```

Update the container:

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

If the container uses the local `jmapi:local` image, rebuild it in `/opt/jmapi-src` before restarting:

```bash
git pull
docker build -t jmapi:local .
cd /opt/jmapi
docker compose up -d
```

Restart:

```bash
docker compose restart jmapi
```

## Security Notes

This service accepts Jimeng/Dreamina session tokens from request headers and uses them to call upstream APIs. Treat every `Authorization` header as a sensitive credential.

The service is best suited for personal, low-frequency use. It depends on unofficial upstream web APIs and browser session tokens, so upstream behavior, token validity, rate limits, and account risk can change without notice.

For private use, add at least one of these controls at the reverse proxy layer:

- IP allowlist.
- Basic Auth.
- VPN-only access.
- Firewall rule that blocks public access to port `5100`.

Avoid enabling `debug: true` in production because request and upstream payload logs can become too verbose.

## Timeout Notes

Image generation can keep the HTTP request open for up to about 30 minutes, and video generation can keep it open for up to about 60 minutes. Keep the reverse proxy timeout above the longest expected request duration:

```nginx
proxy_read_timeout 3700s;
proxy_send_timeout 3700s;
send_timeout 3700s;
```
