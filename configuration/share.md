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

## Two Scopes

`warden share` covers two independent scopes: a global provider that shares projects through Traefik, and a project provider that runs an agent inside a single project's own compose stack. A developer can use either, or both at once.

|                     | Global                                                  | Project                                                          |
|---------------------|----------------------------------------------------------|-------------------------------------------------------------------|
| Selector            | `WARDEN_SHARE_PROVIDER` in `~/.warden/.env`             | `WARDEN_SHARE` in the project `.env` or `.env.local`             |
| Where the agent runs | Container in `warden svc`                               | Service `share` inside the project's own compose stack           |
| Routing             | Traefik labels, matched on `dev.warden.share.domain`     | Agent forwards directly to Varnish (if `WARDEN_VARNISH=1`) or Nginx |
| Credentials         | `~/.warden/.env`                                        | `~/.warden/.env`, never the project `.env`                       |
| Getting the URL     | `TRAEFIK_PUBLIC_DOMAIN` set on the project               | `warden share url`                                                |

Both scopes may be active on the same project at the same time.

## Global Provider Configuration

### Global, Per-Developer (`~/.warden/.env`)

* `WARDEN_SHARE_PROVIDER` selects the global share provider, for example `cloudflared`.

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

## Project Providers

A project provider is selected per project with `WARDEN_SHARE` in the project's `.env` or `.env.local`, for example:

```{code-block} bash
WARDEN_SHARE=quick
```

Setting `WARDEN_SHARE` starts an agent as the `share` service inside that project's own compose stack. The agent forwards traffic it receives to `http://varnish:80` when `WARDEN_VARNISH=1`, otherwise to `http://nginx:80`. There is no Traefik router involved: the agent is the ingress for the project's public hostname.

When a provider needs a developer credential (an auth token, an auth key), that credential lives only in `~/.warden/.env`, never in the project `.env` or `.env.local`. Optional per-project settings for a provider (such as a reserved domain name) do belong in the project `.env`/`.env.local`, since they are not secrets.

`warden share url` prints the current public URL for the project's provider. Some providers only obtain their public hostname after the agent container has started and connected; until then `warden share url` exits with status `1` and a message telling you to check again once the environment is running.

`warden share status` shows the provider name, provider-specific status lines, and the URL if one is available yet.

`warden share update` applies to the global provider only; project providers do not have a persistent ingress configuration to regenerate; running `update` while a project provider is selected fails with a message to that effect.

Global and project scope are independent: a project can have `TRAEFIK_PUBLIC_DOMAIN` set for the global provider and `WARDEN_SHARE` set for a project provider at the same time, giving the project two separate public URLs.

### Cloudflare Quick Tunnel (`WARDEN_SHARE=quick`)

Quick Tunnel needs no Cloudflare account and no configuration. Set the provider and start the environment:

```{code-block} bash
WARDEN_SHARE=quick
```

```{code-block} bash
warden env up
warden share url
```

The public URL is assigned by Cloudflare at each tunnel startup and is random, in the form `https://<random-words>.trycloudflare.com`. `warden share url` reads it from the agent's logs; if it prints nothing yet, wait a few seconds and try again.

There is no restart policy for the Quick Tunnel agent, so any Docker restart of the `share` container gets a new URL. Do not rely on the URL staying the same across restarts, and do not bookmark it for anything long-lived.

Quick Tunnel is intended for development use only. Cloudflare limits it to 200 in-flight requests, and it does not support server-sent events.

### ngrok (`WARDEN_SHARE=ngrok`)

ngrok requires a free ngrok account and an auth token. Get a token from the [ngrok dashboard](https://dashboard.ngrok.com/get-started/your-authtoken) and set it in `~/.warden/.env`:

```{code-block} bash
WARDEN_NGROK_AUTHTOKEN=<your-token>
```

`warden env up` fails with a fatal error naming this variable if it is not set.

Optionally, reserve a free static domain in the ngrok dashboard and set it in the project's `.env` or `.env.local`:

```{code-block} bash
WARDEN_NGROK_DOMAIN=myproject.ngrok-free.app
```

Without `WARDEN_NGROK_DOMAIN` the agent is assigned a new random `*.ngrok-free.app`/`*.ngrok.app` hostname on every start, the same way Quick Tunnel is.

The ngrok free tier includes one static domain, three concurrent online endpoints, 1 GB of bandwidth per month, and 20,000 HTTP(S) requests per month. Requests beyond these limits are refused until the next billing period.

Free-tier traffic through the browser gets an interstitial warning page before reaching the project. This only affects browser navigation; API clients and automated requests can skip it by sending the header `ngrok-skip-browser-warning` with any value.

### Tailscale (`WARDEN_SHARE=tailscale`)

Tailscale shares a project over your own tailnet rather than a public tunnel provider. It requires a reusable, non-ephemeral auth key generated in the Tailscale admin console, set in `~/.warden/.env`:

```{code-block} bash
WARDEN_TAILSCALE_AUTHKEY=<your-auth-key>
```

Use a reusable key, not a one-time key: the agent's node identity is kept in a Docker volume, and a one-time key cannot re-authenticate a container that has already used it once.

By default the project is only reachable from devices on your tailnet, through Tailscale Serve. To expose it publicly through Tailscale Funnel instead, set in the project `.env`/`.env.local`:

```{code-block} bash
WARDEN_TAILSCALE_FUNNEL=1
```

Funnel additionally requires a `nodeAttrs` grant for `funnel` in the tailnet's ACL policy file before any node in the tailnet is allowed to use it, independently of `WARDEN_TAILSCALE_FUNNEL`. Funnel only serves on ports 443, 8443, and 10000.

`WARDEN_TAILSCALE_HOSTNAME` sets the node's hostname on the tailnet; it defaults to the environment name. The resulting public hostname has the form `https://<hostname>.<tailnet>.ts.net`.

Warden generates `.warden/share-tailscale.json` inside the project directory to configure Serve/Funnel for the agent; this file is derived from `WARDEN_TAILSCALE_FUNNEL` and the project's upstream and should not be edited by hand. The agent's node identity is kept in a Docker volume so the hostname stays stable across restarts; removing that volume causes the agent to register as a new node with a new hostname the next time it starts.

### Troubleshooting

- **`warden share url` reports the URL is not available yet**: the agent has not connected yet, or `warden env up` has not been run. Wait a few seconds and try again; if it persists, check `warden env logs share` for errors from the agent itself.
- **`warden env up` fails with a message about a missing ngrok token**: `WARDEN_NGROK_AUTHTOKEN` is not set in `~/.warden/.env`. This is fatal for `up`/`start`; the environment will not come up with `WARDEN_SHARE=ngrok` configured until the token is set.
- **Tailscale Funnel returns 403 or is unavailable**: the tailnet's ACL policy file is missing the `nodeAttrs` grant for `funnel`. Add it in the Tailscale admin console before `WARDEN_TAILSCALE_FUNNEL=1` will work.
- **Tailscale hostname changed unexpectedly**: the agent's state volume was removed or recreated, so it registered as a new node. Recreate the volume consistently across restarts, or accept the new hostname and update wherever it was configured as a base URL.

## Public Hostnames and Magento Base URLs

Magento redirects requests to the store's configured base URL when the incoming `Host` header does not match it (`web/url/redirect_to_base`), and generates links from the configured base URLs rather than from the request's `Host` header. A random hostname from Quick Tunnel or an unreserved ngrok domain will not render the site correctly under this setting: either disable `redirect_to_base` for the store view being tested, or configure a dedicated store view whose base URL matches the hostname you are using. A stable hostname, such as a named Cloudflare Tunnel, an ngrok static domain, or a Tailscale hostname, can be configured as a base URL for a dedicated store view without this per-session friction.

## Adding A Provider

A provider file is added as `utils/share/<name>.sh` and declares its scope at the top with `SHARE_PROVIDER_SCOPE=global` or `SHARE_PROVIDER_SCOPE=project`.

A global provider also ships a `docker-compose.share-<name>.yml` file and implements six functions: `shareProviderIsConfigured` (whether the provider has enough state to run), `shareProviderComposeFile` (path to the compose file to include), `shareProviderPreflight` (warnings shown on `svc up`), `shareProviderRegenerateConfig` (rewrite the provider's configuration from the current domain list), `shareProviderStatus` (output for `warden share status`), and `shareProviderCommand` (handling for any provider-specific subcommands).

A project provider ships a single partial, `environments/includes/share-<name>.base.yml`, defining the `share` service. The partial forwards to `${WARDEN_SHARE_UPSTREAM}`, the upstream Warden resolves to `varnish` or `nginx` depending on the project's configuration, and sets `traefik.enable=false` since a project provider does not route through Traefik. A project provider implements the hooks it needs:

- `shareProviderRequireConfig`: fatal with setup instructions when a required developer credential is missing, a no-op otherwise. Called before the project's compose stack is brought up, and by `warden share` itself.
- `shareProviderPrepare`: optional hook run before compose, for exporting extra `WARDEN_SHARE_*` variables or writing generated files under the project's `.warden/` directory. Defaults to a no-op.
- `shareProviderUrl`: prints the public URL, or returns a non-zero status and prints nothing when it is not available yet.
- `shareProviderStatus`: prints provider-specific status lines for `warden share status`.
- `shareProviderCommand`: handling for provider-specific subcommands, returning `64` for an unknown subcommand.
