# Architecture Overview

Warden runs three layers of services on your development machine:

```{mermaid}
flowchart TB
    classDef default fill:#ffffff,stroke:#3f51b5,stroke-width:1.5px,color:#000000,font-weight:bold,font-size:12px;

    subgraph DevSys ["Developer's System"]
        direction TB

        subgraph Host ["Host Services"]
            direction LR
            h1["Warden CLI"] ~~~ h2["Docker Engine"] ~~~ h3["Mutagen File Sync"] ~~~ h4["Trusted SSL Root CA"]
        end

        subgraph Global ["Global Docker Services — warden svc up"]
            direction LR
            g1["Traefik<br>Reverse Proxy + SSL"] ~~~ g2["DNSmasq"] ~~~ g3["Mailpit<br>Email Catcher"] ~~~ g4["SSH Tunnel"] ~~~ g5["phpMyAdmin"] ~~~ g6["Portainer"]
        end

        subgraph Project ["Per-Project Docker Services — warden env up"]
            direction LR
            p1["Nginx"] ~~~ p2["PHP-FPM"] ~~~ p3["MariaDB / MySQL"] ~~~ p4["Redis / Valkey"] ~~~ p5["Elasticsearch /<br>OpenSearch"] ~~~ p6["RabbitMQ"] ~~~ p7["Varnish"]
        end

        Host ~~~ Global ~~~ Project
    end

    style DevSys fill:#f5f5f5,stroke:#666666,stroke-width:1px,rx:8,ry:8
    style Host fill:transparent,stroke:none
    style Global fill:#fff3e0,stroke:#FF9800,stroke-width:1px,rx:8,ry:8
    style Project fill:#e8f5e9,stroke:#4CAF50,stroke-width:1px,stroke-dasharray: 5 5,rx:8,ry:8
```

## Host Services

The Warden CLI orchestrates everything via Docker Compose. On your host machine:

- **Warden CLI** — Bash tool that manages all Docker services
- **Docker Engine** — Runs all containers
- **Mutagen** — Bi-directional file sync between host and containers (macOS)
- **Trusted SSL Root CA** — Local certificate authority for `*.test` HTTPS domains

## Global Docker Services

Started once with `warden svc up`, shared across all projects:

| Service | Purpose | URL |
|---------|---------|-----|
| Traefik | Reverse proxy + SSL termination | traefik.warden.test |
| DNSmasq | DNS resolver for `*.test` domains | — |
| Mailpit | Email catcher (SMTP) | webmail.warden.test |
| SSH Tunnel | Database access for GUI clients (:2222) | — |
| phpMyAdmin | Database management UI | phpmyadmin.warden.test |
| Portainer | Container management UI (opt-in) | portainer.warden.test |

## Per-Project Docker Services

Started per project with `warden env up`, each on an isolated Docker network:

| Service | Default | Toggle Flag |
|---------|---------|-------------|
| Nginx | All types | `WARDEN_NGINX` |
| PHP-FPM | All types | — |
| MariaDB / MySQL | All types | `WARDEN_DB` |
| Redis / Valkey | All types | `WARDEN_REDIS` / `WARDEN_VALKEY` |
| Elasticsearch / OpenSearch | Magento 2 | `WARDEN_ELASTICSEARCH` / `WARDEN_OPENSEARCH` |
| RabbitMQ | Magento 2 | `WARDEN_RABBITMQ` |
| Varnish | Magento 2 | `WARDEN_VARNISH` |

### Additional Optional Services

These can be enabled per-project via flags in your `.env` file:

- **Blackfire** / **SPX** — PHP profiling (`WARDEN_BLACKFIRE`, `WARDEN_PHP_SPX`)
- **Selenium** — Browser automation testing with optional VNC (`WARDEN_SELENIUM`)
- **Allure** — Test report dashboard (`WARDEN_ALLURE`)
- **PHP-Debug** — Xdebug-enabled PHP-FPM (used via `warden debug`)
- **ElasticHQ** — Elasticsearch management UI (`WARDEN_ELASTICHQ`)
