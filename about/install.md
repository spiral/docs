# Install
You can install Spiral components independently or use pre-build application skeleton which enables most of the framework 
functions. You can either downgrade Web skeleton or use CLI skeleton with minimal number dependencies to start a new application.

Default Web (full-stack) skeleton is available on https://github.com/spiral/app

Installation includes a customized version of RoadRunner application server with additional extensions and framework-specific functions enabled.
<br/>

Server Requirements
--------
Make sure that your server is configured with following PHP version and extensions:
* PHP 7.2+, 64bit
* *mb-string* extension (spiral is UTF-8 centric framework)
* PDO Extension with desired database drivers

Web Application Bundle
--------
Application bundle includes the following components:
* High-Performance HTTP, HTTP/2 server based on [RoadRunner](https://roadrunner.dev)
* Console commands via Symfony/Console
* Queue support for AMQP, Beanstalk, Amazon SQS, in-Memory
* Stempler template engine
* Translation support by Symfony/Translation
* Security, validation, filter models
* PSR-7 HTTP pipeline, session, encrypted cookies
* DBAL and migrations support
* Monolog, DotEnv
* [Cycle DataMapper ORM](https://github.com/cycle)

Installation
--------
```
composer create-project spiral/app
```

> Application server will be downloaded automatically (`php-curl` and `php-zip` required).

Once the application is installed you can ensure that it was configured properly by executing:

```
$ php ./app.php configure
```

To start application server execute:

```
$ ./spiral serve -v -d
```

On Windows:

```$xslt
$ spiral.exe serve -v -d
```

The application will be available on `http://localhost:8080`.

> Read more about application server configuration [here](https://roadrunner.dev/docs).

## Other Skeletons
Check other skeleton builds:
- https://github.com/spiral/app-cli - minimal CLI build
- https://github.com/spiral/app-grpc - GRPC specific build (no views, HTTP)
