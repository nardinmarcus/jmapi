---
name: namoo-jmapi
description: Operate a local Jimeng or Dreamina API gateway for image and video generation. Use this skill when Codex needs to generate images, compose images, generate videos, check Jimeng or Dreamina account credits, claim daily credits, handle multiple session tokens with credit-aware rotation, or call the OpenAI-compatible endpoints exposed by the jmapi service.
---

# Jimeng API

## Purpose

Use this skill to work with the local `jmapi` service. The service wraps Jimeng or Dreamina browser-session APIs and exposes OpenAI-compatible endpoints for image and video generation.

Assume the API base URL is `http://localhost:5100` unless the user provides another URL.

## Safety Rules

- Treat every Jimeng or Dreamina session token as a secret.
- Never hardcode tokens in this skill, scripts, commits, examples, or logs.
- Do not expose this service publicly without a separate gateway secret.
- Prefer low-frequency personal use. Upstream behavior, account risk, rate limits, token validity, and credit rules may change.
- When generated images are downloaded by `scripts/generate_image.py`, report saved file paths and stop. Do not open, inspect, or analyze generated images unless the user explicitly asks.

## Local Credentials

Keep publishable skill files and local credentials separate:

- Commit `namoo-jmapi/.env.example` with placeholder variable names only.
- Store real local values in `namoo-jmapi/.env.local`; this file is ignored by Git and must not be committed.
- Prefer environment variables or `namoo-jmapi/.env.local` over command-line secrets, because shell history may retain command arguments.
- Use `JMAPI_KEY` only for deployments protected by the `X-JMAPI-Key` gateway header. Local `localhost:5100` usually does not need it.

Local-only credential file:

```bash
cp namoo-jmapi/.env.example namoo-jmapi/.env.local
```

Expected variables:

```bash
JMAPI_BASE_URL=http://localhost:5100
JMAPI_KEY=replace_with_gateway_key
JIMENG_SESSION_ID=token1,token2
```

The bundled script automatically loads `namoo-jmapi/.env.local` when present. Explicit command-line arguments still work when needed.

For manual `curl` calls, load the same local file into the shell first:

```bash
set -a
source namoo-jmapi/.env.local
set +a
```

## Token Format

Use the request header:

```bash
Authorization: Bearer token1,token2,token3
```

Token variants:

- Domestic Jimeng: raw `sessionid`.
- Dreamina US: prefix with `us-`.
- Dreamina HK: prefix with `hk-`.
- Dreamina JP: prefix with `jp-`.
- Dreamina SG: prefix with `sg-`.
- Token-bound proxy: put the proxy before the token with `@`.

Example:

```bash
Authorization: Bearer socks5://127.0.0.1:1080@us-token1,hk-token2,token3
```

The service performs credit-aware token selection for image and video generation. It checks tokens in order, skips unavailable tokens, attempts daily credit receive when a token has zero credit, and uses the first token that becomes available.

## Basic Checks

Check service liveness:

```bash
curl -sS http://localhost:5100/ping
```

Check account validity:

```bash
curl -sS -X POST http://localhost:5100/token/check \
  -H "Content-Type: application/json" \
  -d '{"token":"TOKEN"}'
```

Check credits for one or more tokens:

```bash
curl -sS -X POST http://localhost:5100/token/points \
  -H "Authorization: Bearer token1,token2"
```

Claim daily credits for one or more tokens:

```bash
curl -sS -X POST http://localhost:5100/token/receive \
  -H "Authorization: Bearer token1,token2"
```

## Image Generation

Prefer the bundled script for image generation because it calls the API, downloads returned URLs immediately, and saves the generated files.

Save output according to the user's request:

- If the user specifies an output path or folder, pass it with `--output-dir`.
- If the user does not specify an output path, save to the current project's `pic/` directory by omitting `--output-dir`.
- Always report the saved local file paths after the script finishes.
- Do not leave the user with only temporary image URLs unless they explicitly asked for raw API output.

Use only parameters the user explicitly requested. Let script defaults apply when the user did not specify model, ratio, resolution, or other optional settings.

Text-to-image:

```bash
python namoo-jmapi/scripts/generate_image.py text \
  "a precise visual prompt"
```

Image-to-image composition:

```bash
python namoo-jmapi/scripts/generate_image.py image \
  "transform the input into a clean product render" \
  --images "/absolute/path/input.png"
```

Text-to-image with user-specified output folder:

```bash
python namoo-jmapi/scripts/generate_image.py text \
  "a precise visual prompt" \
  --output-dir "/absolute/path/to/output"
```

Optional parameters, only when user-requested:

```bash
python namoo-jmapi/scripts/generate_image.py text \
  "wide cinematic city skyline" \
  --model "jimeng-4.5" \
  --ratio "16:9" \
  --resolution "2k"
```

Script dependencies:

```bash
python -m pip install requests Pillow
```

## Direct API Calls

Use direct calls when the user needs raw API behavior, base64 output, video generation, or custom request bodies.

For protected remote deployments, include the gateway key from the local environment:

```bash
-H "X-JMAPI-Key: $JMAPI_KEY"
```

Text-to-image:

```bash
curl -sS -X POST http://localhost:5100/v1/images/generations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $JIMENG_SESSION_ID" \
  -d '{"model":"jimeng-4.5","prompt":"a precise visual prompt"}'
```

Image composition:

```bash
curl -sS -X POST http://localhost:5100/v1/images/compositions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $JIMENG_SESSION_ID" \
  -d '{"model":"jimeng-4.5","prompt":"combine these references","images":["https://example.com/input.png"]}'
```

Video generation:

```bash
curl -sS -X POST http://localhost:5100/v1/videos/generations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $JIMENG_SESSION_ID" \
  -d '{"model":"jimeng-video-seedance-2.0-fast","prompt":"a short cinematic motion shot","ratio":"16:9","duration":5}'
```

## Model and Parameter Discipline

- Do not invent `ratio`, `resolution`, `duration`, `model`, `negative_prompt`, or `sample_strength` unless the user requested them.
- Use defaults for unspecified image parameters.
- For `nanobanana`, remember that regional behavior differs:
  - `us-` tokens are effectively fixed at `1024x1024`.
  - `hk-`, `jp-`, and `sg-` tokens force lower resolution but may allow ratios.
  - Domestic tokens should use Jimeng models instead.
- `intelligent_ratio` is useful only for compatible Jimeng image models. Do not add it unless requested.

## Troubleshooting

- If credits are insufficient, first call `/token/points`, then `/token/receive`.
- If one account is out of credits, pass multiple tokens in the same `Authorization` header; the service rotates to the first account with usable credit.
- If a region returns auth or upload errors, verify the token prefix and whether that account requires a bound proxy.
- If generated URLs expire before download, rerun generation and download immediately.
- If local API calls fail, confirm the service is running and that the configured API URL matches the active port.
