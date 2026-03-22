# Cloudflare Tunnel

Warden integrates with [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) to expose local development environments to the public internet. This is useful for:

- Sharing work-in-progress with clients or teammates
- Testing webhooks from external services
- Mobile device testing on real domains
- QA review without deploying to staging

## Prerequisites

- A [Cloudflare account](https://dash.cloudflare.com/sign-up)
- A domain configured in Cloudflare (DNS managed by Cloudflare)

## Setup

### 1. Authenticate with Cloudflare

```{code-block} bash
warden cf login
```

This opens your browser for Cloudflare authentication. A `cert.pem` file is saved to `~/.warden/etc/cloudflared/`.

### 2. Create a tunnel

```{code-block} bash
warden cf create
```

This creates a tunnel named `warden` (or pass a custom name: `warden cf create mytunnel`). The tunnel ID is saved to `~/.warden/.env` as `WARDEN_CLOUDFLARED_TUNNEL_ID`.

### 3. Start global services

```{code-block} bash
warden svc up
```

The `cloudflared` container starts automatically when `WARDEN_CLOUDFLARED_TUNNEL_ID` is set.

### 4. Configure a project

Add `TRAEFIK_PUBLIC_DOMAIN` to your project's `.env`:

```{code-block} bash
TRAEFIK_PUBLIC_DOMAIN=myproject.example.com
```

### 5. Configure DNS in Cloudflare

In the [Cloudflare dashboard](https://dash.cloudflare.com/), add a CNAME record pointing your domain to the tunnel:

| Type | Name | Target |
|------|------|--------|
| CNAME | myproject | `<tunnel-id>.cfargotunnel.com` |
| CNAME | *.myproject | `<tunnel-id>.cfargotunnel.com` |

Replace `<tunnel-id>` with your tunnel ID from `warden cf status`.

### 6. Start the project

```{code-block} bash
warden env up
```

Warden automatically regenerates the cloudflared config to include your project's domain. Your site is now accessible at `https://myproject.example.com`.

## How It Works

```
Internet → Cloudflare Edge → cloudflared container → Traefik → nginx → php-fpm
```

1. **cloudflared** maintains a persistent connection to Cloudflare's edge network
2. Incoming requests for your domain are forwarded through the tunnel to the local **cloudflared** container
3. **cloudflared** routes the request to **Traefik** (the local reverse proxy)
4. **Traefik** matches the `Host` header to the correct project and forwards to **nginx**
5. **nginx** serves the request from **php-fpm** as usual

The cloudflared configuration is automatically regenerated whenever you run `warden env up`, `warden env down`, `warden env start`, or `warden env stop`.

## Configuration

### Global (`~/.warden/.env`)

| Variable | Description |
|----------|-------------|
| `WARDEN_CLOUDFLARED_TUNNEL_ID` | Tunnel UUID. Set automatically by `warden cf create`. When present, enables the cloudflared service. |

### Project (`.env`)

| Variable | Description |
|----------|-------------|
| `TRAEFIK_PUBLIC_DOMAIN` | Public domain for this project (e.g., `myproject.example.com`). When set, the project is exposed via the Cloudflare Tunnel. |

## CLI Reference

| Command | Description |
|---------|-------------|
| `warden cf login` | Authenticate with Cloudflare (opens browser) |
| `warden cf create [name]` | Create a new tunnel (default name: `warden`) |
| `warden cf delete` | Delete the tunnel from Cloudflare |
| `warden cf status` | Show tunnel status and connected domains |
| `warden cf update` | Force-regenerate cloudflared config and restart |
| `warden cf logout` | Remove all cloudflared credentials and config |

## Troubleshooting

### Tunnel not connecting

1. Check the cloudflared container is running: `warden svc ps`
2. Check logs: `warden svc logs cloudflared`
3. Verify credentials exist: `ls ~/.warden/etc/cloudflared/`

### Domain not routing

1. Check the domain label is set: `docker inspect <nginx-container> | grep cf.domain`
2. Verify the config was generated: `cat ~/.warden/etc/cloudflared/config.yml`
3. Force regenerate: `warden cf update`
4. Check DNS records in Cloudflare dashboard point to `<tunnel-id>.cfargotunnel.com`

### Certificate errors

The cloudflared-to-Traefik connection uses `noTLSVerify: true` because Traefik uses Warden's self-signed certificates. This is expected and only affects the local tunnel segment — the public-facing connection uses Cloudflare's edge certificates.
