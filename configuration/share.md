# Sharing A Project Publicly

`warden share` exposes a running project to the public internet through a share provider, without deploying anywhere. This is useful for:

- Receiving webhooks or payment provider callbacks that need a publicly reachable URL
- Showing work in progress to a client or teammate
- Testing on a real mobile device over a real domain instead of `.test`

## How It Fits Warden's Architecture

A share provider runs as a global service, started with `warden svc up` alongside Traefik and the other {doc}`global services <../services>`. The provider's container joins the `warden` network and forwards incoming public traffic to Traefik, which then routes it exactly the way it routes local `.test` traffic: by matching the `Host` header to the project's Nginx or Varnish container.

```
Internet -> provider edge -> share agent (warden network) -> Traefik -> Nginx/Varnish
```

Because routing happens at the Traefik layer, sharing a project does not change how the project itself is configured. The same container serves both `myproject.test` and the public hostname.

## User Configuration

### Global, Per-Developer (`~/.warden/.env`)

* `WARDEN_SHARE_PROVIDER` selects the share provider, for example `cloudflared`.

This setting belongs in `~/.warden/.env`, not in any project's `.env`. `~/.warden/.env` lives outside every project repository, so the tunnel credentials and tunnel identity it references stay tied to the developer's own account rather than being shared or committed to a project.

### Per Project (`.env`)

* `TRAEFIK_PUBLIC_DOMAIN` is the public hostname for the project, for example `myproject.example.com`.

The value must be a plain hostname: letters, digits and hyphens, with at least one dot. `warden env up` rejects anything else, since the value is used verbatim in a Traefik router rule and in the provider's ingress configuration.

If a developer wants to use a personal hostname without committing it to the project, `TRAEFIK_PUBLIC_DOMAIN` can be set in `.env.local` instead of `.env` (see {doc}`../environments/customizing`).

Projects running Varnish are routed through Varnish automatically; no separate configuration is needed.

## Cloudflare Tunnel Provider

### Prerequisites

- A Cloudflare account
- A domain (zone) on Cloudflare
- Docker

### Setup

1. Set the provider in `~/.warden/.env`:

   ```{code-block} bash
   WARDEN_SHARE_PROVIDER=cloudflared
   ```

2. Authenticate with Cloudflare:

   ```{code-block} bash
   warden share login
   ```

   This opens a browser for Cloudflare authentication. On success a `cert.pem` is stored in `~/.warden/etc/cloudflared/`.

3. Create a tunnel:

   ```{code-block} bash
   warden share create [name]
   ```

   The tunnel name defaults to `warden`. This writes `WARDEN_CLOUDFLARED_TUNNEL_ID` into `~/.warden/.env`, and adds `WARDEN_SHARE_PROVIDER=cloudflared` to that file if it is not already set.

4. Create the DNS record. In the Cloudflare dashboard, add a proxied CNAME record for the hostname (and for `*.hostname` if subdomains are used) pointing to `<tunnel-id>.cfargotunnel.com`. This can also be done with `cloudflared tunnel route dns`. Warden does not create this record.

5. Start global services:

   ```{code-block} bash
   warden svc up
   ```

6. Set the public hostname in the project's `.env`:

   ```{code-block} bash
   TRAEFIK_PUBLIC_DOMAIN=myproject.example.com
   ```

7. Start the project:

   ```{code-block} bash
   warden env up
   ```

8. Check status:

   ```{code-block} bash
   warden share status
   ```

   This prints the configured provider, the tunnel ID, whether the container is running, and the list of domains currently connected.

### Configuration Regeneration

The provider's ingress configuration is regenerated from the set of domains carrying the `dev.warden.share.domain` label on any running container. This happens on every `warden env up`, `warden env down`, `warden env start`, `warden env stop`, and on `warden svc up`. The share agent container is restarted only when the regenerated ingress list actually changed, so starting or stopping one project does not interrupt tunnels for other projects.

`warden share update` forces a regeneration and restart. `warden share delete` removes the tunnel from Cloudflare. `warden share logout` removes all cloudflared credentials and configuration from `~/.warden/etc/cloudflared/`; if a tunnel is still configured it asks for confirmation first, since the tunnel would otherwise be left orphaned on the Cloudflare side.

Cloudflared files live under `~/.warden/etc/cloudflared/`: `cert.pem` (Cloudflare account credential), the tunnel credentials file, and the generated `config.yml`.

:::{note}
The generated tunnel configuration sets `noTLSVerify: true` on the connection between the share agent and Traefik. This is expected: it applies only to the internal Docker network segment, where the agent is trusting Warden's self-signed Traefik certificate. The public-facing leg of the connection is terminated with Cloudflare's own edge TLS certificate.
:::

### Troubleshooting

- **Warning about a missing tunnel on `warden svc up`**: `WARDEN_SHARE_PROVIDER` is set but no tunnel has been created yet. Run `warden share login` and `warden share create`.
- **404 from Cloudflare**: the DNS record is missing, or the project's domain is not in the generated ingress list. Check `warden share status` for the connected domains.
- **`TRAEFIK_PUBLIC_DOMAIN` rejected on `warden env up`**: the value must match a plain hostname pattern (letters, digits, hyphens, at least one dot).

## Adding A Provider

A share provider is added as `utils/share/<name>.sh` plus a `docker-compose.share-<name>.yml` file. The script must implement six functions: `shareProviderIsConfigured` (whether the provider has enough state to run), `shareProviderComposeFile` (path to the compose file to include), `shareProviderPreflight` (warnings shown on `svc up`), `shareProviderRegenerateConfig` (rewrite the provider's configuration from the current domain list), `shareProviderStatus` (output for `warden share status`), and `shareProviderCommand` (handling for any provider-specific subcommands).
