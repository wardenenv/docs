# Customizing An Environment

## Version Customization via `.env`

To configure your project with a non-default PHP version, add the following to the project's `.env` file and run `warden env up` to re-create the affected containers:

    PHP_VERSION=7.2

The versions of Elasticsearch, Varnish, Redis, NodeJS and Composer may also be similarly configured using variables in the `.env` file:

  * `ELASTICSEARCH_VERSION`
  * `REDIS_VERSION`
  * `VALKEY_VERSION`
  * `VARNISH_VERSION`
  * `RABBITMQ_VERSION`
  * `NODE_VERSION`
  * `COMPOSER_VERSION`

## Database

Warden supports both **MariaDB** (default) and **MySQL** as the database engine. The distribution and version are controlled via two environment variables in your project's `.env` file:

  * `MYSQL_DISTRIBUTION` — set to `mariadb` (default) or `mysql`
  * `MYSQL_DISTRIBUTION_VERSION` — the image tag/version to use

For example, to use MariaDB 11.4:

    MYSQL_DISTRIBUTION=mariadb
    MYSQL_DISTRIBUTION_VERSION=11.4

To use MySQL 8.4 instead:

    MYSQL_DISTRIBUTION=mysql
    MYSQL_DISTRIBUTION_VERSION=8.4

:::{note}
Drupal and CakePHP environments use `DB_DISTRIBUTION` and `DB_DISTRIBUTION_VERSION` instead of the `MYSQL_` prefix. The values and behavior are the same.
:::

After changing the database distribution or version, run `warden env up` to re-create the database container.

:::{warning}
Switching distributions (e.g. from MariaDB to MySQL) is **not** a drop-in replacement for existing data volumes. If you change the distribution on an existing environment, you should export your database first, remove the db volume (`warden env down -v`), and re-import after the new container is running.
:::

Start of some environments could be skipped by using variables in `.env` file:

  * `WARDEN_DB=0`
  * `WARDEN_REDIS=0`
  * `WARDEN_VALKEY=0`

## Docker Compose Overrides

For customizations beyond `.env` variables, Warden supports per-project Docker Compose override files. Create a `.warden/warden-env.yml` file in your project root to override or extend the default service configuration.

This file is a standard Docker Compose YAML partial. Warden loads it **after** all built-in service definitions, giving it the highest merge precedence — any service, volume, label, or environment variable defined here will override the defaults.

:::{tip}
Preview the fully merged Docker Compose configuration at any time with:

    warden env config
:::

### OS-Specific Overrides

In addition to `.warden/warden-env.yml`, Warden also loads OS-specific override files if they exist:

  * `.warden/warden-env.darwin.yml` — applied only on **macOS**
  * `.warden/warden-env.linux.yml` — applied only on **Linux**

This is useful for platform-specific volume mount options or performance tuning.

### Custom Docker Image

To use a custom PHP-FPM image (e.g., with pre-installed extensions or from a private registry):

```yaml
services:
  php-fpm:
    image: my-registry/custom-php:8.3
  php-debug:
    image: my-registry/custom-php:8.3-debug
```

### Adding Extra Services

You can add services not included by default. For example, to add Adminer with Traefik routing:

```yaml
services:
  adminer:
    image: adminer:latest
    labels:
      - traefik.enable=true
      - traefik.http.routers.${WARDEN_ENV_NAME}-adminer.rule=Host(`adminer.${TRAEFIK_DOMAIN}`)
      - traefik.http.routers.${WARDEN_ENV_NAME}-adminer.tls=true
```

After adding this and running `warden env up -d`, Adminer will be accessible at `https://adminer.<project>.test`.

## PHP Specific Customizations

To override default PHP settings, mount a custom `php.ini` file via `.warden/warden-env.yml`:

```yaml
services:
  php-fpm:
    volumes:
      - ./.warden/php/zz-config.ini:/etc/php.d/zz-config.ini
  php-debug:
    volumes:
      - ./.warden/php/zz-config.ini:/etc/php.d/zz-config.ini
```

Now add the referenced `.warden/php/zz-config.ini` file with your wanted changes.
For example you could change the error reporting value:

```ini
error_reporting=(E_ALL ^ E_DEPRECATED)
```

The config file will be merged with the default `01-php.ini` resp. override the values
included there. The default values are the following:

```ini
date.timezone = UTC
max_execution_time = 3600
max_input_vars = 10000
memory_limit = 2G
display_errors = On
post_max_size = 25M
session.auto_start = Off
upload_max_filesize = 25M
```

## Nginx Specific Customizations

To override the default nginx configuration of your project, add the following to `.warden/warden-env.yml`:

```yaml
services:
  nginx:
    volumes:
      - ./.warden/nginx/custom.conf:/etc/nginx/default.d/custom.conf
```

There you can specify a custom Nginx configuration which will be included following the `.conf` files within the `/etc/nginx/available.d` directory: `include /etc/nginx/default.d/*.conf`

## Advanced: Project-Level Environment Partials

Beyond `.warden/warden-env.yml`, Warden supports a more granular override mechanism through project-level environment partial directories. For each service partial (e.g., `php-fpm`, `nginx`, `db`), Warden searches the following locations in order:

  1. Built-in includes: `<warden>/environments/includes/`
  2. Built-in type-specific: `<warden>/environments/<env-type>/`
  3. User home includes: `~/.warden/environments/includes/`
  4. User home type-specific: `~/.warden/environments/<env-type>/`
  5. **Project includes: `.warden/environments/includes/`**
  6. **Project type-specific: `.warden/environments/<env-type>/`**

Files at each location use suffixes: `.base.yml` (all platforms), `.darwin.yml` (macOS), `.linux.yml` (Linux).

For example, to override just the `php-fpm` partial for a Magento 2 project, create:

    .warden/environments/includes/php-fpm.base.yml

This is loaded in addition to (and after) the built-in `php-fpm.base.yml`, so Docker Compose merge semantics apply.

:::{note}
The `.warden/warden-env.yml` file is still loaded **after** all partials, so it always has the final say.
:::

## Multiple Domains

For configuring multiple top-level domains, see [Multiple Domains](../configuration/multipledomains.md).

## Magento 1 Specific Customizations

If you use a `modman` structure, initialize the environment in your project path.
The `.modman` folder and the corresponding `.basedir` file will be recognized and set up automatically.

## Magento 2 Specific Customizations

:::{note}

The split database feature has been deprecated as of Adobe Commerce 2.4.2 and is not supported on Adobe Commerce cloud infrastructure.

::::

The following variables can be added to the project's `.env` file to enable additional database containers for use with the Magento 2 (Commerce Only) [split-database solution](https://experienceleague.adobe.com/en/docs/commerce-operations/configuration-guide/storage/split-db/multi-master).

  * `WARDEN_SPLIT_SALES=1`
  * `WARDEN_SPLIT_CHECKOUT=1`

Start of some Magento 2 specific environments could be skipped by using variables in `.env` file:

  * `WARDEN_ELASTICSEARCH=0`
  * `WARDEN_VARNISH=0`
  * `WARDEN_RABBITMQ=0`
