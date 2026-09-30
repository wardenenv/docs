# Secrets Providers

Development projects often need a handful of secrets that should not sit in plaintext in the project's `.env` file: a `COMPOSER_AUTH` token, a licence key, or a sandbox API key. Warden can read those values from a secrets manager and inject them into the PHP containers on `warden env up`.

Secrets injection is opt-in. Set `WARDEN_SECRETS` to a provider name, and that provider is asked for the values; when `WARDEN_SECRETS` is unset, nothing changes and the environment behaves exactly as before.

## How it works

```{mermaid}
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant Core as Warden core
    participant CLI as Provider CLI (op / vault)
    participant Compose as Docker Compose
    participant PHP as PHP containers

    Dev->>Core: warden env up
    Core->>CLI: read secrets
    CLI-->>Core: KEY=VALUE lines
    Core->>Core: filter reserved names
    Core->>Compose: compose up with a names-only override,<br>values exported to this process only
    Compose->>PHP: container environment
    Note over Core: override removed on exit
```

- Injection and re-reading happen only on `warden env up`, and never for an environment of type `local`. `warden env start` and `warden env restart` reuse the existing containers, and a container keeps the environment it was created with, so the values are still there; they are just not read from the provider again.
- The `php-fpm` and `php-debug` containers always receive the variables. When `WARDEN_PHP_SPX`, `WARDEN_BLACKFIRE` or `WARDEN_MAGENTO2_GRAPHQL_SERVER` is enabled, the matching `php-spx`, `php-blackfire` or `php-graphql` container receives them too.
- `warden shell` inherits the variables from the running container, so they are available in a shell without extra setup.
- Every `warden env up` reads the secrets again, so a rotated value arrives on the next `warden env up`.
- The values are exported into the `docker compose` process, so they are also available for `${VAR}` interpolation in the compose files, including the project's `.warden/warden-env.yml`, on the first and every later `warden env up`.

## Choosing a provider

| Provider | `WARDEN_SECRETS` value | Required variables | Optional variables | CLI needed |
|----------|------------------------|--------------------|--------------------|------------|
| [1Password](https://developer.1password.com/docs/cli/) | `1password` | `WARDEN_OP_ENVIRONMENT_ID` or a `.env.op` file | `WARDEN_OP_ACCOUNT` | `op` (2.33.0-beta.02 or later for Environments) |
| [HashiCorp Vault](https://developer.hashicorp.com/vault) | `vault` | `WARDEN_VAULT_PATH` | `WARDEN_VAULT_MOUNT` (default `secret`), `WARDEN_VAULT_ADDR`, `WARDEN_VAULT_NAMESPACE` | `vault` and `jq` |

## 1Password

The 1Password provider has two modes. Pick one per project: a 1Password Environment, or secret references listed in a `.env.op` file.

```{mermaid}
flowchart TD
    B{"WARDEN_OP_ENVIRONMENT_ID set?"} -->|yes| C{".env.op present?"}
    C -->|yes| D["warning: use one of them"]
    C -->|no| E["Environments mode"]
    B -->|no| F{".env.op present?"}
    F -->|yes| G["Items mode"]
    F -->|no| H["warning: set one"]
```

| Mode | Configured by | 1Password side | CLI call | Best for |
|------|---------------|----------------|----------|----------|
| Environments | `WARDEN_OP_ENVIRONMENT_ID` | A variable set managed in 1Password | `op environment read --no-masking` | A variable set a team manages in one place |
| Items | a `.env.op` file | Individual fields of existing vault items | `op read` | Picking specific fields from existing items, with the references reviewed in git |

### Environments

Add the following to the project's `.env` file:

```bash
WARDEN_SECRETS=1password
WARDEN_OP_ENVIRONMENT_ID=<environment_id>
```

Values are injected as 1Password returns them.

Warden does not read a 1Password credential from the project `.env`. The `op` CLI authenticates on its own, using the signed-in desktop app integration or a session token from `op signin`. On a host signed in to more than one 1Password account, set `WARDEN_OP_ACCOUNT` to the account to read from:

```bash
WARDEN_OP_ACCOUNT=<account>
```

1Password Environments require 1Password CLI 2.33.0-beta.02 or later. See the [1Password Environments documentation](https://developer.1password.com/docs/environments/) for creating an environment and adding variables to it.

### Items (secret references)

Add the following to the project's `.env` file:

```bash
WARDEN_SECRETS=1password
```

and a `.env.op` file in the project root next to `.env` with the references:

```
COMPOSER_AUTH=op://project-x/composer/composer-auth
NEW_RELIC_KEY="op://project-x/New Relic/license key"
```

- The `.env.op` file holds references only, so it is meant to be committed.
- Each reference is resolved with `op read` on `warden env up`, and the field's content becomes the variable's value. See the [secret reference syntax](https://developer.1password.com/docs/cli/secret-reference-syntax/).
- If one reference cannot be read, the whole injection is skipped.
- Lines that are not `KEY=op://…` are ignored and reported by line number.
- Values with line breaks are skipped with a warning, so a multi-line field such as a private key cannot be injected this way.
- `WARDEN_OP_ACCOUNT` applies to every reference.
- `WARDEN_OP_ENVIRONMENT_ID` must be unset to use this mode.

## HashiCorp Vault

Add the following to the project's `.env` file:

```bash
WARDEN_SECRETS=vault
WARDEN_VAULT_PATH=myproject/development
```

The remaining settings are optional:

```bash
WARDEN_VAULT_MOUNT=secret
WARDEN_VAULT_ADDR=https://vault.example.com:8200
WARDEN_VAULT_NAMESPACE=admin
```

- `WARDEN_VAULT_MOUNT` defaults to `secret`.
- `WARDEN_VAULT_ADDR` and `WARDEN_VAULT_NAMESPACE` default to whatever the Vault CLI is already configured with.
- The secret path is relative to the mount, may not contain `..`, and may not start with `/`.

Warden does not read a Vault credential from the project `.env`. The `vault` CLI authenticates on its own, using the host's `VAULT_TOKEN` or the token helper configured after `vault login`.

Both KV v1 and KV v2 secrets engines are supported. Warden reads the pairs from `.data.data` for KV v2 and from `.data` for KV v1.

Values of type object, array or null, and strings that contain a line break, cannot be injected. They are skipped and the skipped names are listed in a warning. Keys that are not valid shell identifiers (letters, digits and underscores, not starting with a digit) are dropped silently, because they cannot be used as environment variable names.

The `jq` command is required alongside the `vault` CLI.

## Reserved variables

Warden refuses to inject some names, so a secrets source cannot change how the stack is built. There are two groups.

Managed by Warden: the variables Warden uses to build the stack: `CHOWN_DIR_LIST`, `COMPOSER_MEMORY_LIMIT`, `COMPOSER_VERSION`, `HISTFILE`, `NODE_VERSION`, `SSH_AUTH_SOCK`, `SSH_AUTH_SOCK_PATH_ENV`, `XDEBUG_VERSION`, plus every name under the `WARDEN_`, `TRAEFIK_` or `PHP_` prefixes.

Would change the host process that runs `docker compose`: `PATH`, `HOME`, `USER`, `LOGNAME`, `SHELL`, `PWD`, `OLDPWD`, `IFS`, `TMPDIR`, `ENV`, `BASH_ENV`, `SHELLOPTS`, `BASHOPTS`, `PS4`, `CDPATH`, `GLOBIGNORE`, `TERM`, `LANG`, plus every name under the `DOCKER_`, `COMPOSE_`, `BUILDKIT_`, `BUILDX_`, `LD_`, `DYLD_`, `BASH_FUNC_`, `LC_`, `OP_`, `VAULT_` or `MUTAGEN_` prefixes.

When a provider returns a reserved name, Warden warns `Variables reserved by Warden were not injected: <names>` and starts the environment without it.

## Security notes

:::{important}
- Warden writes no secret values to disk. The temporary compose override holds variable names only, and it is removed when `warden env up` exits.
- The values are exported only to the `docker compose` process that starts the environment. Once the containers run, the values live in the container environment and are visible to anyone who can run `docker inspect` on the host, exactly like any other compose `environment:` entry, so treat this as central storage and rotation of development secrets, not as container isolation.
- Names that would steer that `docker compose` process itself, such as `PATH` and the Docker client and loader variables, are refused outright.
- Provider output is parsed, never evaluated, so a value cannot execute a command on the host.
- The secrets manager credentials themselves never come from the project `.env`; the provider CLI authenticates on its own.
- Keep development secrets separate from production ones.
:::

## Troubleshooting

Every failure below is a warning. `warden env up` still starts the environment; it just starts without the secrets.

| Warning | Cause | Fix |
|---------|-------|-----|
| `1Password CLI (op) could not be found; …` / `Vault CLI (vault) could not be found; …` / `jq could not be found; …` | The provider CLI, or `jq` for Vault, is not on the `PATH`. | Install the missing tool and re-run `warden env up`. |
| `Unknown secrets provider '<name>' in WARDEN_SECRETS. Available providers: …; skipping secret injection.` | `WARDEN_SECRETS` names a provider that has no `utils/secrets/<name>.sh` file. | Use one of the listed provider names. |
| `WARDEN_SECRETS must be a provider name (lowercase letters, digits, dashes). …` | `WARDEN_SECRETS` contains characters outside lowercase letters, digits and dashes. | Correct the value. |
| `WARDEN_OP_ENVIRONMENT_ID is not a valid 1Password Environment ID; …` / `WARDEN_VAULT_PATH is not a valid Vault secret path; …` | The ID or path fails the provider's validation. The same pattern applies to `WARDEN_VAULT_MOUNT`, `WARDEN_VAULT_NAMESPACE` and `WARDEN_VAULT_ADDR`. | Fix the value in the project `.env`. |
| `Both WARDEN_OP_ENVIRONMENT_ID and .env.op are configured; use one of them. Skipping secret injection.` | A 1Password Environment ID and a `.env.op` file are both present. | Keep one mode and remove the other. |
| `Set WARDEN_OP_ENVIRONMENT_ID or add a .env.op file with secret references; skipping secret injection.` | Neither 1Password mode is configured. | Set the environment ID, or add `.env.op`. |
| `Could not read 1Password Environment '<id>' (Environments require 1Password CLI 2.33.0-beta.02 or later); …` / `Could not read 1Password secret reference for <KEY>; …` / `Could not read Vault secret '<mount>/<path>'; …` | The provider CLI is not signed in, lacks permission, or cannot reach the service. | Sign in with the provider CLI, check access, then re-run. |
| `1Password returned masked values; skipping secret injection.` | The `op` CLI ignored `--no-masking` and returned concealed values, for example an older beta. | Update the 1Password CLI to 2.33.0-beta.02 or later. |
| `Ignored .env.op lines that are not KEY=op://… references: <line numbers>` | `.env.op` contains lines that are not `KEY=op://…` references. | Fix or remove those lines; they are not injected. |
| `1Password values with line breaks cannot be injected and were skipped: <names>` | A referenced field's value contains a line break. | Store a single-line value in that field, or inject it another way. |
| `Variables reserved by Warden were not injected: <names>` | The provider returns one or more names Warden reserves, because they build the stack or steer the `docker compose` process. | Rename the variable in the secrets manager. |
| `Secrets provider '<provider>' returned no variables; …` / `Secrets provider '<provider>' defines only variables reserved by Warden; …` | The provider returned an empty result, or only reserved names. | Check the environment ID or secret path, and that it holds injectable variables. |

## Writing a provider

A provider is a shell file at `utils/secrets/<name>.sh` that implements the two functions from the core contract:

```{mermaid}
flowchart LR
    A["core sources utils/secrets/PROVIDER.sh"] --> B["secretsProviderRequireConfig<br>validate configuration"]
    B --> C["secretsProviderRead<br>fetch secrets"]
    C --> D["KEY=VALUE lines on stdout"]
    D --> E["core filters and injects"]
```

A minimal provider looks like this:

```bash
#!/usr/bin/env bash
[[ ! ${WARDEN_DIR} ]] && >&2 echo -e "\033[31mThis script is not intended to be run directly!\033[0m" && exit 1

function secretsProviderRequireConfig {
    if [[ ! "${WARDEN_EXAMPLE_PATH:-}" =~ ^[A-Za-z0-9_./-]+$ ]]; then
        warning "WARDEN_EXAMPLE_PATH is not a valid path; skipping secret injection."
        return 1
    fi
}

function secretsProviderRead {
    if ! command -v example >/dev/null 2>&1; then
        warning "Example CLI (example) could not be found; skipping secret injection."
        return 1
    fi

    example read "${WARDEN_EXAMPLE_PATH}"
}
```

Rules for a provider:

- Prefix the provider's own settings with `WARDEN_<NAME>_` so they cannot collide with Warden's own variables.
- Never read the secrets manager credentials from the project `.env`; let the provider CLI authenticate on its own.
- On failure, print a warning that names the problem and return non-zero. The core then skips injection and the environment starts without secrets.
- Never print secret values to stderr or write them to disk; the only output that carries values is the `KEY=VALUE` lines on stdout.
