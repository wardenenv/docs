# Warden Usage

## Environment Management

Starting a stopped environment:

    warden env start

Stopping a running environment:

    warden env stop

Display the resolved environment configuration in `docker-compose` format:

    warden env config

Tail environment nginx and php logs:

    warden env logs --tail 0 -f nginx php-fpm php-debug

Remove volumes completely:

    warden env down -v

## Shell Access

Launch a shell session within the project environment's `php-fpm` container:

    warden shell

Launch a debug-enabled shell (Xdebug) session:

    warden debug

## Database

Import a database (if you don't have `pv` installed, use `cat` instead):

    pv /path/to/dump.sql.gz | gunzip -c | warden db import

Monitor database processlist:

    watch -n 3 "warden db connect -A -e 'show processlist'"

## Redis / Valkey

Connect to redis/valkey:

    warden redis
    warden valkey

Flush redis/valkey completely:

    warden redis flushall
    warden valkey flushall

Run redis continuous stat mode:

    warden redis --stat

## Varnish

Tail the varnish activity log:

    warden env exec -T varnish varnishlog

Flush varnish:

     warden env exec -T varnish varnishadm 'ban req.url ~ .'

## Troubleshooting

Warden troubleshooting and debug information:

    warden doctor

    # Verbose mode also print environment variables along with debug and configuration information.
    warden doctor -v

## Further Information

Run `warden help` and `warden env -h` for more details and useful command information.
