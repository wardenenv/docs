# Ngrok

For information on what Ngrok is, please see the [what is Ngrok](https://ngrok.com/docs/what-is-ngrok/) in Ngrok documentation.


## Configuring a new project using NGROK

### 1. Enable ngrok in warden
Ngrok may be enabled on all env types by adding the following to the project's `.env` file (or exporting them to environment variables prior to starting the environment):

```
WARDEN_NGROK=1

NGROK_AUTHTOKEN=<authtoken>
```

Note: You can obtain the token used in the above from within your Ngrok account under your ngrok [Dashboard](https://dashboard.ngrok.com/) > Click Your [Authtoken](https://dashboard.ngrok.com/get-started/your-authtoken).

### 2. Initialize the ngrok configuration file : 

```shell
warden ngrok init domain1.test [domain2.test ...]
```

You will obtain two files : 
 - ${WARDEN_ENV_PATH}/.warden/ngrok.yml  : The ngrok configuration (see [Documentation](https://ngrok.com/docs/agent/config/))
```
version: "2"
log: stdout
tunnels:
    domain1.test:
       proto: "http"
       addr: "caddy-ngrok:2080"
    domain2.test:
       proto: "http"
       addr: "caddy-ngrok:2080"

```
 - ${WARDEN_ENV_PATH}/.warden/ngrok.caddy : The caddy configuration
```
{
   order replace after encode
}
:2080 {
    @get {
        method GET
        path /*
    }
    handle @get {
        replace {
        }
    }

    reverse_proxy varnish:80 {
        header_up Accept-Encoding identity
        header_up X-Forwarded-Proto https
    }
}
```
> Note that the **${WARDEN_ENV_PATH}/.warden/ngrok.caddy is empty** on first initialization,
> and configuration should be refreshed on every restart of your docker environment as the ngrok urls will change on free plans.


### 3. Refresh the caddy ngrok configuration file :

Caddy is used to automatically rewrite urls in from the ngrok to your local environment urls in order to avoid having to change any url configuration on your local environment.

```shell
warden ngrok refresh-config
```

You will obtain a `ngrok.caddy` file like this one : 
```
{
   order replace after encode
}
:2080 {
    @get {
        method GET
        path /*
    }
    handle @get {
        replace {
            domain2.test 5eb079afae74.ngrok.app
            domain1.test 497b85f07962.ngrok.app
        }
    }

    reverse_proxy varnish:80 {
        header_up Accept-Encoding identity
        header_up X-Forwarded-Proto https
        header_up * 5eb079afae74\.ngrok\.app  domain2.test
        header_down * domain2\.test 5eb079afae74.ngrok.app
        header_up * 497b85f07962\.ngrok\.app  domain1.test
        header_down * domain1\.test 497b85f07962.ngrok.app
    }
}
```

> After each restart of the ngrok container, you will have to refresh the caddy configuration to reflect ngrok urls changes.