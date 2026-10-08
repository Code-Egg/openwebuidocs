---
sidebar_position: 203
title: "HTTPS using OpenLiteSpeed"
---

# HTTPS Using OpenLiteSpeed

OpenLiteSpeed (OLS) can serve Open WebUI over HTTPS and forward requests to the application running on your server or in Docker. This guide covers two setups: installing OLS on the host with its one-click script, or running the proxy in Docker Compose.

Choose one setup method. Both use AutoSSL to request and renew a trusted certificate for your domain.

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
    # Keep your existing image, volumes, environment, and other settings.
    networks:
      - ls-net

networks:
  ls-net:
    name: ls-net
    external: true
```

If the service already has a `networks` entry, add `ls-net` to that list rather than replacing its existing networks. The proxy will use `open-webui:8080` as its backend address.

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

The proxy project publishes the web ports. Confirm that ports `80` and `443` are allowed through your firewall and that the domain resolves to this server.

## Verify HTTPS

Open `https://chat.example.com` in a browser. Open WebUI should load with a valid certificate. Sign in and send a test prompt to confirm that API requests and streamed responses work through the proxy.

After verification, make sure the backend port is not reachable from the public internet. With Docker, the proxy and backend can communicate over `ls-net` without publishing the backend port to the host. If you need a host port for local access, restrict it to `127.0.0.1` or block public access with your firewall.

## Optional security settings

OpenLiteSpeed docker also offers features such as OWASP protection, CAPTCHA, per-client throttling, access control, realms, and security headers. Configure these through the proxy environment file or the WebAdmin Console as appropriate for your setup. Test each setting with Open WebUI before enabling it in production, since some rules can interfere with legitimate application requests.

