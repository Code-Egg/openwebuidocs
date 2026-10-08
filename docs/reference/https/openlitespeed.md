---
sidebar_position: 203
title: "HTTPS using OpenLiteSpeed"
---

# HTTPS Using OpenLiteSpeed

OpenLiteSpeed (OLS) can serve Open WebUI over HTTPS and forward requests to the application running on your server or in Docker. This guide covers running the openlitespeed as a reverse proxy,  auto apply SSL and enable security features in Docker Compose.

## Prerequisites

- A domain name with DNS `A` and/or `AAAA` records pointing to the server.
- Public access to ports `80` and `443` for certificate validation and HTTPS traffic.
- Open WebUI running and reachable by the proxy.
- Docker Engine and the Compose plugin for the Docker Compose method.

:::warning

Do not expose the Open WebUI backend port publicly. Once the proxy is working, allow public traffic only through the ports used by OpenLiteSpeed, typically `80` and `443`.

:::

## Run the proxy in Docker Compose

The OpenLiteSpeed proxy container and Open WebUI container must share a Docker network so the proxy can reach the backend by service name.

### Create the shared network

Create the external network once:

```bash
docker network inspect ls-net >/dev/null 2>&1 || docker network create ls-net
```

In the Open WebUI Compose project, attach the `open-webui` service to this network. Add the following network declaration at the end of the Compose file:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    volumes:
      - open-webui:/app/backend/data
    #environment:
    #  - WEBUI_SECRET_KEY=your-secret-key
    restart: unless-stopped

volumes:
  open-webui:

networks:
  default:
    name: ls-net
    external: true
```

### Configure the OLS proxy container

Clone the [OpenLiteSpeed proxy Docker Compose project](https://github.com/litespeedtech/ols-proxy-docker-env) and create its environment file:

```bash
git clone https://github.com/litespeedtech/ols-proxy-docker-env
cd ols-proxy-docker-env
cp .env.example .env
```

Edit `.env` and set these values. Use your own domain and an email address for certificate notices:

```dotenv
OLS_IMAGE=litespeedtech/openlitespeed:latest
BACKEND_IP=open-webui
BACKEND_PORT=8080
DOMAIN=chat.example.com
PROXY_METHOD=context
PROXY_SOCKET=true
ACME_EMAIL=admin@example.com
```

Start the proxy:

```bash
docker compose up -d
```

## Verify HTTPS

Open `https://chat.example.com` in a browser. Open WebUI should load with a valid certificate. Sign in and send a test prompt to confirm that API requests and streamed responses work through the proxy.

After verification, make sure the backend port is not reachable from the public internet. With Docker, the proxy and backend can communicate over `ls-net` without publishing the backend port to the host. If you need a host port for local access, restrict it to `127.0.0.1` or block public access with your firewall.

## Optional security settings

OpenLiteSpeed docker also offers features such as OWASP protection, CAPTCHA, per-client throttling, access control, realms, and security headers. Configure these through the proxy environment file or the WebAdmin Console as appropriate for your setup. Test each setting with Open WebUI before enabling it in production, since some rules can interfere with legitimate application requests.

Example security content in the .env file
```
### Global security controls. These apply to every mapped domain and cannot be set in domains.conf.
THROTTLING=false
RECAPTCHA=false
MODSECURITY=false

### Per-client throttling values used only when THROTTLING=true.
THROTTLING_STATIC_REQ_PER_SEC=1000
THROTTLING_DYNAMIC_REQ_PER_SEC=50
THROTTLING_OUT_BANDWIDTH=0
THROTTLING_IN_BANDWIDTH=0
THROTTLING_SOFT_LIMIT=50
THROTTLING_HARD_LIMIT=100
THROTTLING_BLOCK_BAD_REQUEST=true
THROTTLING_GRACE_PERIOD=15
THROTTLING_BAN_PERIOD=60

### CAPTCHA values used only when RECAPTCHA=true. Provider keys are optional.
### RECAPTCHA_TYPE: checkbox, invisible, or hcaptcha.
RECAPTCHA_TYPE=checkbox
RECAPTCHA_SITE_KEY=
RECAPTCHA_SECRET_KEY=
RECAPTCHA_MAX_TRIES=10
RECAPTCHA_ALLOWED_ROBOT_HITS=100
RECAPTCHA_CONNECTION_LIMIT=100
RECAPTCHA_SSL_CONNECTION_LIMIT=100

### OWASP CRS is baked into the image. Change the version only with: docker compose up -d --build
OWASP_CRS_VERSION=4.21.0
```
