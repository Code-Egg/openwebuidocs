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

#### Step 1. Download the repository and enter its directory:

Clone the [OpenLiteSpeed proxy Docker Compose project](https://github.com/litespeedtech/ols-proxy-docker-env) and create its environment file:

```bash
git clone https://github.com/litespeedtech/ols-proxy-docker-env
cd ols-proxy-docker-env
cp .env.example .env
```

#### Step 2. Open `.env` in a text editor and configure the deployment:

Edit `.env` and set these values. Use your own domain and an email address for certificate notices:

```dotenv
OLS_IMAGE=litespeedtech/openlitespeed:latest
BACKEND_IP=open-webui
BACKEND_PORT=8080
DOMAIN=www.example.com
PROXY_METHOD=context
PROXY_SOCKET=true
ACME_EMAIL=admin@example.com
```

    | Variable | Description |
    | --- | --- |
    | `OLS_IMAGE` | OpenLiteSpeed container image to run. |
    | `BACKEND_IP` | Backend container name or IP. |
    | `BACKEND_PORT` | Port on which the application listens inside its container. |
    | `DOMAIN` | Domain name served by OpenLiteSpeed. |
    | `PROXY_METHOD` | Method used to configure the OpenLiteSpeed reverse proxy. Supports `context` and `rewrite` values. |
    | `PROXY_SOCKET` | Enables WebSocket proxying when set to `true`. |
    | `ACME_EMAIL` | Email address used for ACME certificate registration and notifications. |


#### Step 3. Start OpenLiteSpeed proxy

```bash
docker compose up -d
```


## Verify HTTPS

Open `https://www.example.com` in a browser. Open WebUI should load with a valid certificate. Sign in and send a test prompt to confirm that API requests and streamed responses work through the proxy.

After verification, make sure the backend port is not reachable from the public internet. With Docker, the proxy and backend can communicate over `ls-net` without publishing the backend port to the host. If you need a host port for local access, restrict it to `127.0.0.1` in the compose file or block public access with your firewall.

## Optional security settings

OpenLiteSpeed docker also offers features such as OWASP protection, CAPTCHA, per-client throttling, access control, realms, and security headers. Configure these through the proxy environment file or the WebAdmin Console as appropriate for your setup. Test each setting with Open WebUI before enabling it in production, since some rules can interfere with legitimate application requests.

Example default security config settings in the .env file
```
### Global security controls. These apply to every mapped domain and cannot be set in domains.conf.
THROTTLING=false
RECAPTCHA=false
MODSECURITY=false
```

To enable Throttling, set THROTTLING in `.env` to `true`:
```
THROTTLING=true
```
Recreate the proxy container to apply the change:
```
docker compose up -d --force-recreate
```