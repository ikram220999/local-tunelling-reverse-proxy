# Uplinkr

Expose a local HTTP port to the internet via a secure tunnel — no port forwarding, no firewall rules.

```
uplinkr http 3000
# → https://amber-brook.example.com
```

---

## How it works

```
Browser → Nginx (TLS, wildcard) → Bridge Server → WebSocket → uplinkr CLI → localhost:3000
```

1. The CLI opens a persistent WebSocket connection to the **Bridge Server**, registering its tunnel ID.
2. Nginx routes incoming HTTPS requests for `<subdomain>.example.com` to the Bridge Server, passing the tunnel ID via `X-Tunnel-Id`.
3. The Bridge Server serialises the HTTP request as JSON and sends it over the WebSocket to the matching CLI instance.
4. The CLI fetches the request against `localhost:<port>` and sends the response back over the same WebSocket.

---

## Components

| Directory | Language | Role |
|---|---|---|
| `websocketclient/` | Node.js / React (Ink) | CLI — proxies traffic over a WebSocket tunnel |
| `websocketserver/` | C# / .NET 10 | Bridge Server — relays HTTP↔WebSocket |
| `nginx-reverseproxy/` | Nginx | Reverse proxy — TLS termination, wildcard subdomain routing |
| `website/` | HTML/CSS/JS | Marketing landing page |
| `fallback-page/` | HTML | Custom nginx error pages (404, 503, etc.) |
| `core-api/` | C# / .NET 10 | ⚠️ Work in progress — not part of the current system, ignore for now |

---

## Prerequisites

- Node.js ≥ 18
- .NET 10 SDK
- Docker
- Nginx with a wildcard TLS certificate for your domain

---

## Quick start

### 1. Configure

**`websocketclient/config.js`** — set your server URLs:
```js
export const WS_SERVER_URL = "wss://ws-server.example.com";
export const BASE_DOMAIN    = "example.com";
```

### 2. Start the Bridge Server

```bash
cd websocketserver
dotnet run
# Listens on http://localhost:4001
```

### 3. Start Nginx

```bash
cd nginx-reverseproxy
docker compose up -d
```

### 4. Install and run the CLI

```bash
npm install -g uplinkr

uplinkr http 3000
```

---

## CLI usage

```
uplinkr http <port> [--log <path>]
```

| Argument | Description |
|---|---|
| `<port>` | Local port to expose |
| `--log <path>` | Directory to write request/response logs |

**Examples:**
```bash
uplinkr http 5000
uplinkr http 8080 --log ./logs
```

The CLI prints a live traffic log to the terminal — method, path, status, and latency for every request.

---

## Nginx configuration

The `nginx-reverseproxy/` directory contains:

| File | Purpose |
|---|---|
| `nginx.conf` | Main nginx config — includes `sites-enabled/*` and `tunnels/*` |
| `uplinkr` | Virtual host for the root domain (redirects HTTP → HTTPS) |
| `uplinkr-wildcard` | Wildcard virtual host — routes `*.example.com` to the Bridge Server |
| `uplinkr-ws-server` | Virtual host for the WebSocket server endpoint |
| `fallback-page/` | Custom error pages served by nginx on 404 / 503 / 5xx |

---

## Project structure

```
.
├── websocketserver/        # Bridge Server (.NET 10)
├── websocketclient/        # CLI (Node.js)
│   ├── cli.jsx             # Entry point
│   ├── app.jsx             # Ink UI
│   └── bridge.js           # WebSocket bridge client
├── nginx-reverseproxy/     # Nginx config + Docker Compose
│   └── fallback-page/      # Error pages
├── website/                # Landing page
└── core-api/               # ⚠️ Ignore — work in progress, not part of the current system
```
