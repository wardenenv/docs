# Installing Magento 2

The below example demonstrates the from-scratch setup of the Magento 2 application for local development. A similar process can easily be used to configure an environment of any other type. This assumes that Warden has been previously started via `warden svc up` as part of the installation procedure.

:::{note}
In addition to the below manual process, there is a `Github Template available for Magento 2 <https://github.com/wardenenv/warden-env-magento2>`_ allowing for quick setup of new Magento projects. To use this, click the green "Use this template" button to create your own repository based on the template repository, run the init script and update the README with any project specific information.
:::

1. Create a new directory on your host machine at the location of your choice and then jump into the new directory to get started:

        mkdir -p ~/Sites/exampleproject
        cd ~/Sites/exampleproject

2. From the root of your new project directory, run `env-init` to create the `.env` file with configuration needed for Warden and Docker to work with the project.

        warden env-init exampleproject magento2

    The result of this command is a `.env` file in the project root (tip: commit this to your VCS to share the configuration with other team members) having the following contents:

        WARDEN_ENV_NAME=exampleproject
        WARDEN_ENV_TYPE=magento2
        WARDEN_WEB_ROOT=/

        TRAEFIK_DOMAIN=exampleproject.test
        TRAEFIK_SUBDOMAIN=app

        WARDEN_DB=1
        WARDEN_ELASTICSEARCH=0
        WARDEN_OPENSEARCH=1
        WARDEN_ELASTICHQ=0
        WARDEN_VARNISH=1
        WARDEN_RABBITMQ=1
        WARDEN_REDIS=1

        OPENSEARCH_VERSION=2.12
        MYSQL_DISTRIBUTION=mariadb
        MYSQL_DISTRIBUTION_VERSION=10.6
        NODE_VERSION=20
        COMPOSER_VERSION=2
        PHP_VERSION=8.3
        PHP_XDEBUG_3=1
        RABBITMQ_VERSION=3.13
        REDIS_VERSION=7.2
        VARNISH_VERSION=7.5

        WARDEN_SYNC_IGNORE=

        WARDEN_ALLURE=0
        WARDEN_SELENIUM=0
        WARDEN_SELENIUM_DEBUG=0
        WARDEN_BLACKFIRE=0
        WARDEN_SPLIT_SALES=0
        WARDEN_SPLIT_CHECKOUT=0
        WARDEN_TEST_DB=0
        WARDEN_MAGEPACK=0

        MAGEPACK_VERSION=2.11

        BLACKFIRE_CLIENT_ID=
        BLACKFIRE_CLIENT_TOKEN=
        BLACKFIRE_SERVER_ID=
        BLACKFIRE_SERVER_TOKEN=

    :::{note}
    Starting with Magento 2.4.8, set:

        WARDEN_REDIS=0
        WARDEN_VALKEY=1
        VALKEY_VERSION=8.1

    :::

3. Sign an SSL certificate for use with the project (the input here should match the value of `TRAEFIK_DOMAIN` in the above `.env` example file):

        warden sign-certificate exampleproject.test

4. Next you'll want to start the project environment:

        warden env up

    :::{warning}
    If you encounter an error about ``Mounts denied``, follow the instructions in the error message and run ``warden env up`` again.
    :::

5. Drop into a shell within the project environment. Commands following this step in the setup procedure will be run from within the `php-fpm` docker container this launches you into:

        warden shell

6. Configure global Magento Marketplace credentials

        composer global config http-basic.repo.magento.com <username> <password>

    :::{note}
    To locate or generate a new set of composer credentials for Adobe Commerce (Magento 2) you will need an Adobe Commerce Marketplace account,
    instructions on the Adobe site: [Get your authentication keys](https://experienceleague.adobe.com/en/docs/commerce-operations/installation-guide/prerequisites/authentication-keys).

    If you have previously configured global credentials, you may skip this step, as ``~/.composer/`` is mounted into the container from the host machine in order to share composer cache between projects, and also shares the global ``auth.json`` from the host machine.

    Use the **Public key** as your username and the **Private key** as your password.
    :::

7. Initialize project source files using composer create-project and then move them into place:

        META_PACKAGE=magento/project-community-edition META_VERSION=2.4.x

        composer create-project --repository-url=https://repo.magento.com/ \
            "${META_PACKAGE}" /tmp/exampleproject "${META_VERSION}"

        rsync -a /tmp/exampleproject/ /var/www/html/
        rm -rf /tmp/exampleproject/

8. Install the application and you should be all set:

    :::{note}
    Please note that if using Valkey instead of Redis, the value of the session and cache flags will be different
    :::

        ## Install Application
        bin/magento setup:install \
            --backend-frontname=backend \
            --amqp-host=rabbitmq \
            --amqp-port=5672 \
            --amqp-user=guest \
            --amqp-password=guest \
            --db-host=db \
            --db-name=magento \
            --db-user=magento \
            --db-password=magento \
            --search-engine=opensearch \
            --opensearch-host=opensearch \
            --opensearch-port=9200 \
            --opensearch-index-prefix=magento2 \
            --opensearch-enable-auth=0 \
            --opensearch-timeout=15 \
            --http-cache-hosts=varnish:80 \
            --session-save=redis \
            --session-save-redis-host=redis \
            --session-save-redis-port=6379 \
            --session-save-redis-db=2 \
            --session-save-redis-max-concurrency=20 \
            --cache-backend=redis \
            --cache-backend-redis-server=redis \
            --cache-backend-redis-db=0 \
            --cache-backend-redis-port=6379 \
            --page-cache=redis \
            --page-cache-redis-server=redis \
            --page-cache-redis-db=1 \
            --page-cache-redis-port=6379

        ## Configure Application
        bin/magento config:set --lock-env web/unsecure/base_url \
            "https://${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}/"

        bin/magento config:set --lock-env web/secure/base_url \
            "https://${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}/"

        bin/magento config:set --lock-env web/secure/offloader_header X-Forwarded-Proto

        bin/magento config:set --lock-env web/secure/use_in_frontend 1
        bin/magento config:set --lock-env web/secure/use_in_adminhtml 1
        bin/magento config:set --lock-env web/seo/use_rewrites 1

        bin/magento config:set --lock-env system/full_page_cache/caching_application 2
        bin/magento config:set --lock-env system/full_page_cache/ttl 604800

        bin/magento config:set --lock-env catalog/search/enable_eav_indexer 1

        bin/magento config:set --lock-env dev/static/sign 0

        bin/magento deploy:mode:set -s developer
        bin/magento cache:disable block_html full_page

        bin/magento indexer:reindex
        bin/magento cache:flush

10. Generate an admin user and configure 2FA for OTP

         ## Generate localadmin user
         ADMIN_PASS="$(pwgen -n1 16)"
         ADMIN_USER=localadmin

         bin/magento admin:user:create \
             --admin-password="${ADMIN_PASS}" \
             --admin-user="${ADMIN_USER}" \
             --admin-firstname="Local" \
             --admin-lastname="Admin" \
             --admin-email="${ADMIN_USER}@example.com"
         printf "u: %s\np: %s\n" "${ADMIN_USER}" "${ADMIN_PASS}"

         ## Configure 2FA provider
         OTPAUTH_QRI=
         # Python 2: TFA_SECRET=$(python -c "import base64; print base64.b32encode('$(pwgen -A1 128)')" | sed 's/=*$//')
         # Python 3:
         TFA_SECRET=$(python3 -c "import base64; print(base64.b32encode(bytearray('$(pwgen -A1 128)', 'ascii')).decode('utf-8'))" | sed 's/=*$//')
         OTPAUTH_URL=$(printf "otpauth://totp/%s%%3Alocaladmin%%40example.com?issuer=%s&secret=%s" \
             "${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}" "${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}" "${TFA_SECRET}"
         )

         bin/magento config:set --lock-env twofactorauth/general/force_providers google
         bin/magento security:tfa:google:set-secret "${ADMIN_USER}" "${TFA_SECRET}"

         printf "%s\n\n" "${OTPAUTH_URL}"
         printf "2FA Authenticator Codes:\n%s\n" "$(oathtool -s 30 -w 10 --totp --base32 "${TFA_SECRET}")"

         segno "${OTPAUTH_URL}" -s 4 -o "pub/media/${ADMIN_USER}-totp-qr.png"
         printf "%s\n\n" "https://${TRAEFIK_SUBDOMAIN}.${TRAEFIK_DOMAIN}/media/${ADMIN_USER}-totp-qr.png?t=$(date +%s)"

     :::{warning}
     **Adobe Commerce 2.4.8 core issue with ``--lock-env``**

     Until [this core issue](https://github.com/magento/magento2/issues/39836) is addressed, the following command will result in a broken (unable to run ``bin/magento`` and ``mr`` commands) environment:    
     ``bin/magento config:set --lock-env twofactorauth/general/force_providers google``

     The issue is due to the ``DuoSecurity`` 2FA Provider improperly reading the ``force-providers`` value from ``env.php``. 

     The workaround is to, after running the command, manually adjust the following content in ``env.php``:
     ````
     'twofactorauth' => [
       'general' => [
         'force_providers' => [
           'google'
         ]
       ]
     ]
     ````
     with:
     ````
     'twofactorauth' => [
       'general' => [
         'force_providers' => 'google'
       ]
     ]
     ````
     :::

     :::{note}
     Use of 2FA is mandatory on Magento ``2.4.x`` and setup of 2FA should be skipped when installing ``2.3.x`` or earlier. Where 2FA is setup manually via UI upon login rather than using the CLI commands above, the 2FA configuration email may be retrieved from `the Mailhog service <https://mailhog.warden.test/>`_.
     :::

11. Launch the application in your browser:

    - [https://app.exampleproject.test/](https://app.exampleproject.test/)
    - [https://app.exampleproject.test/backend/](https://app.exampleproject.test/backend/)
    - [https://rabbitmq.exampleproject.test/](https://rabbitmq.exampleproject.test/)
    - [https://elasticsearch.exampleproject.test/](https://elasticsearch.exampleproject.test/)

:::{note}
To completely destroy the ``exampleproject`` environment we just created, run ``warden env down -v`` to tear down the project's Docker containers, volumes, etc.
:::
