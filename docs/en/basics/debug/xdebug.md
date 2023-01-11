# Debug and Profiling - XDebug

It is possible to debug the Spiral application like any other classic PHP applications using the xDebug extension.

## IDE Configuration

Read more about the IDE configuration [here](https://roadrunner.dev/docs/php-debugging).

## On-Demand

It is more convenient to start RoadRunner with xDebug enabled only when it's needed. Add the following env variables to
`.rr.yaml` to properly configure xDebug:

```yaml
env:
  PHP_IDE_CONFIG: serverName=application.loc
  XDEBUG_CONFIG: remote_host=localhost max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
``` 

> **Note**
> Alter values according to your environment.

To enable xDebug, run the application server with `-o` (overwrite flag) for the required service:

```bash
./rr serve -o "server.command=php -d zend_extension=xdebug app.php"
```

## In Docker

To alter workers config in docker, use the following or similar config for your container:

```yaml
event-service:
  build:
    dockerfile: Dockerfile
    context: .
  command:
    - /usr/local/bin/rr
    - serve
    - -o
    - server.command=php -d zend_extension=xdebug.so app.php
  environment:
    PHP_IDE_CONFIG: serverName=application.loc
    XDEBUG_CONFIG: remote_host=host.docker.internal max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```
