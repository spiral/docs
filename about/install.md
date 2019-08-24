# Install
You can install Spiral components independently or use pre build application skeleton which enables most of framework 
functions. You can either downgrade Web skeleton or use CLI skeleton with minimal dependencies.

Installation includes customized version of RoadRunner application server with additional extensions and framework 
specific functions enabled.

<br/>

Server Requirements
--------
Make sure that your server is configured with following PHP version and extensions:
* PHP 7.1+, 64bit
* MbString Extension
* PDO Extension with desired database drivers

Application Bundle
--------
Application bundle includes following components:
* High-Performance HTTP, HTTP/2 server based on [RoadRunner](https://roadrunner.dev)
* Console commands via symfony/console
* Queue support for AMQP, Beanstalk, Amazon SQS, in-Memory
* Twig template engine
* Translation support by symfony/translation
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

Once application is installed you can ensure that it was configured properly by executing:

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

Application will be available on `http://localhost:8080`.

> Read more about application server configuration [here](https://roadrunner.dev/docs).

## Other Skeletons
Check other skeleton builds:
    * https://github.com/spiral/app-cli - minimal CLI build
    * https://github.com/spiral/app-grpc - grpc specific build (no views, http)