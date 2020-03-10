# Debug and Profiling - XDebug
It is possible to debug the Spiral application like any other classic PHP applications using xDebug extension.

## IDE Configuration
Read more about IDE configuration [here](https://roadrunner.dev/docs/php-debugging).

## On-Demand
Is it more convenient to start RoadRunner with xDebug enabled only when it's needed. Add the following env variables to
`.rr.yaml` to properly configure xDebug:

```yaml
env:
  PHP_IDE_CONFIG: serverName=application.loc
  XDEBUG_CONFIG: remote_host=localhost max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
``` 

> Alter values according to your environment.

To enable xDebug run application server with `-o` (overwrite flag) for needed service:

```bash
$ ./spiral serve -v -d -o "http.workers.command=php -d zend_extension=xdebug app.php"
```

## In Docker
To alter workers config in docker user the following or similar config for your container:

```yaml
event-service:
    build:
      dockerfile: Dockerfile
      context: .
    command:
    - /usr/local/bin/spiral
    - serve
    - -v
    - -d
    - -o
    - http.workers.command=php -d zend_extension=xdebug.so app.php
    environment:
      PHP_IDE_CONFIG: serverName=application.loc
      XDEBUG_CONFIG: remote_host=host.docker.internal max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```
