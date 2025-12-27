# Cloudflare Lab — WSL as Origin (FastAPI + Cloudflare Tunnel)

This repository documents how to set up a **local WSL machine as a private origin**
behind **Cloudflare**, using **Cloudflare Tunnel** and **FastAPI**.

This lab is designed to safely test:

- DNS
- TLS / SSL modes
- WAF rules
- Redirect & transform rules
- Cache behavior
- Request headers and edge → origin flow

All testing is isolated to a **subdomain** and does **not impact production**.

---

## Architecture Overview

```

Client (Browser / curl)
|
v
Cloudflare Edge

* DNS
* TLS
* WAF (later)
* Rules
  |
  v
  Cloudflare Tunnel (mTLS)
  |
  v
  WSL (Ubuntu 24.04)
* FastAPI origin (localhost:8000)

```

Key properties:

- No public IP exposed
- No port forwarding
- TLS terminates at Cloudflare edge
- Origin reachable only via tunnel

---

## Prerequisites

### Local system

- Windows with **WSL2**
- Ubuntu **24.04 LTS**
- Internet access

### Cloudflare

- Cloudflare account
- Domain already added to Cloudflare

Example used in this lab:

```

lab.decodestrength.com

````

---

## Phase 1 — Origin Setup (WSL)

### 1. Create project directory

```bash
mkdir ~/cf-lab-origin
cd ~/cf-lab-origin
````

---

### 2. Initialize project (uv)

```bash
uv init
uv venv
uv add fastapi uvicorn
```

---

### 3. FastAPI application

Create `main.py`:

```python
from fastapi import FastAPI, Request, HTTPException
from datetime import datetime
import socket
import time

app = FastAPI(title="Cloudflare Lab Origin")

@app.get("/")
async def root():
    return {
        "status": "ok",
        "origin": "wsl-fastapi",
        "hostname": socket.gethostname(),
        "timestamp": datetime.utcnow().isoformat() + "Z"
    }

@app.get("/headers")
async def headers(request: Request):
    return {
        "client": request.client.host if request.client else None,
        "headers": dict(request.headers)
    }

@app.get("/slow")
async def slow():
    time.sleep(5)
    return {"status": "slow response (5s)"}

@app.get("/error")
async def error():
    raise HTTPException(status_code=500, detail="simulated 500")
```

---

### 4. Run the origin

```bash
uv run uvicorn main:app --host 0.0.0.0 --port 8000
```

---

### 5. Local verification

```bash
curl http://127.0.0.1:8000/
curl http://127.0.0.1:8000/headers | jq .
time curl http://127.0.0.1:8000/slow
curl -i http://127.0.0.1:8000/error
```

Expected results:

* `/` → 200 OK
* `/headers` → request headers
* `/slow` → ~5s delay
* `/error` → HTTP 500

---

## Phase 2 — Cloudflare Tunnel

### 1. Install cloudflared (Ubuntu 24.04)

```bash
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /etc/apt/keyrings/cloudflare-main.gpg >/dev/null

echo "deb [signed-by=/etc/apt/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared noble main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update
sudo apt install -y cloudflared
```

Verify:

```bash
cloudflared --version
```

---

### 2. Authenticate cloudflared

```bash
cloudflared tunnel login
```

This opens a browser window to authorize your Cloudflare account.

Verify:

```bash
ls ~/.cloudflared
```

Expected:

```
cert.pem
```

---

### 3. Create tunnel

```bash
cloudflared tunnel create lab-origin
```

Verify:

```bash
cloudflared tunnel list
```

---

### 4. Tunnel configuration

Create `~/.cloudflared/config.yml`:

```yaml
tunnel: lab-origin
credentials-file: /home/expert/.cloudflared/<TUNNEL-UUID>.json

ingress:
  - hostname: lab.decodestrength.com
    service: http://localhost:8000
  - service: http_status:404
```

Validate:

```bash
cloudflared tunnel ingress validate
```

---

### 5. Bind DNS (automatic)

```bash
cloudflared tunnel route dns lab-origin lab.decodestrength.com
```

This automatically creates a proxied DNS record pointing to the tunnel.

---

### 6. Run the tunnel

Ensure FastAPI is running, then:

```bash
cloudflared tunnel run lab-origin
```

Expected behavior:

* Tunnel connects to multiple Cloudflare POPs
* QUIC protocol is used
* Origin remains private

---

## Phase 2 — Verification

### HTTPS access

```bash
curl https://lab.decodestrength.com/
```

---

### Header inspection (Cloudflare → origin)

```bash
curl https://lab.decodestrength.com/headers | jq .
```

Expected headers include:

* `cf-ray`
* `cf-connecting-ip`
* `cf-ipcountry`
* `x-forwarded-proto`
* `cf-visitor`

---

### TLS inspection

```bash
openssl s_client -connect lab.decodestrength.com:443
```

Expected:

* Cloudflare-issued certificate
* TLS 1.3
* Valid certificate chain

---

## Day-2 Operations

### Start lab (another day)

```bash
# Terminal 1 — origin
cd ~/cf-lab-origin
uv run uvicorn main:app --host 0.0.0.0 --port 8000

# Terminal 2 — tunnel
cloudflared tunnel run lab-origin
```

---

### Stop lab

Press `Ctrl + C` in both terminals.

DNS and tunnel configuration remain intact.

---

### Useful commands

```bash
cloudflared tunnel list
cloudflared tunnel info lab-origin
curl -I https://lab.decodestrength.com/
```

---

## Security Notes

* Never expose the origin directly
* Never commit the following files:

  * `~/.cloudflared/cert.pem`
  * `~/.cloudflared/<UUID>.json`
* Always scope Cloudflare rules to:

```
host == "lab.decodestrength.com"
```

---

```

