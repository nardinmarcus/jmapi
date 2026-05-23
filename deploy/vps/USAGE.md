# JMAPI Usage

This guide assumes the VPS deployment is available at:

```text
https://jmapi.namooca.com
```

## Shell Setup

Keep gateway and Jimeng/Dreamina credentials in environment variables:

```bash
export JMAPI_BASE_URL="https://jmapi.namooca.com"
export JMAPI_KEY="REPLACE_WITH_GATEWAY_KEY"
export JIMENG_SESSION_ID="REPLACE_WITH_SESSION_ID"
```

The gateway key is checked by OpenResty through `X-JMAPI-Key`. The Jimeng/Dreamina session token is passed to the application through `Authorization: Bearer`.

## Health Check

`/ping` is intentionally public so uptime monitors can use it without credentials:

```bash
curl -sS "$JMAPI_BASE_URL/ping"
```

Expected response:

```text
pong
```

## Check Credits

```bash
curl -sS \
  -H "X-JMAPI-Key: $JMAPI_KEY" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $JIMENG_SESSION_ID" \
  -X POST "$JMAPI_BASE_URL/token/points"
```

## Generate Images

Start with `1k` for low-cost validation:

```bash
curl -sS \
  -H "X-JMAPI-Key: $JMAPI_KEY" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $JIMENG_SESSION_ID" \
  -d '{
    "model": "jimeng-4.5",
    "prompt": "a quiet cyberpunk street at night, cinematic lighting",
    "ratio": "16:9",
    "resolution": "1k",
    "response_format": "url"
  }' \
  "$JMAPI_BASE_URL/v1/images/generations"
```

Returned image URLs are signed temporary URLs. Download or archive useful results before they expire.

## Token-Level Proxy

If an account needs a specific outbound IP, bind a proxy to that token:

```bash
export JIMENG_SESSION_ID="socks5://127.0.0.1:1080@us-REPLACE_WITH_SESSION_ID"
```

Multiple tokens can be passed as a comma-separated list:

```bash
export JIMENG_SESSION_ID="socks5://127.0.0.1:1080@us-TOKEN1,hk-TOKEN2,TOKEN3"
```

## Update the VPS

After publishing a new image tag:

```bash
ssh myvps
cd /opt/jmapi
docker compose pull
docker compose up -d
docker compose ps
curl -sS https://jmapi.namooca.com/ping
```

## Rotate the Gateway Key

Generate a new key:

```bash
openssl rand -base64 32 | tr -d '=+/' | cut -c1-32
```

Update the OpenResty rule in:

```text
/opt/1panel/www/conf.d/jmapi.conf
```

Then reload OpenResty:

```bash
docker exec 1Panel-openresty-I5lO openresty -t
docker exec 1Panel-openresty-I5lO openresty -s reload
```

## Troubleshooting

Inspect the app container:

```bash
cd /opt/jmapi
docker compose ps
docker logs --tail 100 jmapi
```

Inspect OpenResty:

```bash
docker exec 1Panel-openresty-I5lO openresty -t
tail -100 /opt/1panel/apps/openresty/openresty/log/error.log
```
