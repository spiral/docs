# Getting started — Installation

You can install Spiral components independently or use a
pre-build [application skeleton](https://github.com/spiral/app), which enables most of the framework functions. You can 
either downgrade Web skeleton or use the CLI skeleton with minimal number dependencies to start a new application.

Installation includes the following packages out of the box and framework-specific functions enabled:

- [roadrunner-bridge](https://github.com/spiral/roadrunner-bridge)
- [cycle-bridge](https://github.com/spiral/cylce-bridge)
- [spiral/nyholm-bridge](https://github.com/spiral/nyholm-bridge)

<br/>

Server Requirements
--------
Make sure that your server is configured with the following PHP version and extensions:

* PHP 8.1+, 64bit
* *mb-string* extension (spiral is UTF-8 centric framework)
* *socket* extension

## Installation


```terminal
composer create-project spiral/app my-app
```

> **Note**
> Application server will be downloaded automatically. `php-curl` and `php-zip` required.

Once the application is installed, you can ensure that it has been configured properly by executing:

```terminal
php app.php configure
```

To start application server execute:

:::: tabs

::: tab Linux

Use the following command to run RoadRunner server on **Linux**

```terminal
./rr serve
```

> **Warning**
> Make sure that `rr` binary is executable.

:::

::: tab Windows
Use the following command to run RoadRunner server on **Windows**

```terminal
./rr.exe serve
```
:::

::::

The application will be available on `http://localhost:8080`.

> **Note**
> Read more about application server configuration [here](https://roadrunner.dev/docs).