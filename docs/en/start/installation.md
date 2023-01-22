# Getting started — Installation

The Spiral Framework provides a convenient application installer that allows developers to easily install and configure
the framework and any desired packages through a command line interface. This helps streamline the process of getting
started with Spiral and simplifies the setup of the development environment.

It can be very useful for installing all required config files, environment variables, and bootloaders for the desired
packages and components. The installer allows developers to easily install a variety of packages and components,
including:

- [Cycle bridge](../basics/orm.md)
- [Validators](../validation/factory.md)
- [Event Dispatcher](../advanced/events.md)
- [RoadRunner Bridge](../start/server.md)
- Temporal Bridge
- [Cron jobs scheduler](../advanced/scheduler.md)
- [Sentry bridge](../basics/errors.md)
- [Stempler](../views/stempler.md) or [Twig](../views/twig.md) template engine
- [Serializer](../advanced/serializer.md)
- [Mailer component](../advanced/sendit.md)
- [RoadRunner metrics](../advanced/prometheus-metrics.md)

Having these packages and components readily available and properly configured through the installer can save developers
time and effort in setting up their development environment and getting their projects up and running.

## Server Requirements

Make sure that your server is configured with the following PHP version and extensions:

* PHP 8.1+, 64bit
* *mb-string* extension (spiral is UTF-8 centric framework)
* *socket* extension
* *curl* extension
* *zip* extension

## Installation

Installation process with the installer is very straightforward and easy to use. You can use the following command to
create a new project:

```terminal
composer create-project spiral/app my-app
```

And you will see the following output:

```output
[32mCreating a "spiral/app" project at "./my-app"[39m
[32mInstalling spiral/app (1.1.1)[39m
  - Installing [32mspiral/app[39m ([33m1.1.1[39m): Extracting archive
[32mCreated project in /var/www/my-app[39m
> Installer\Installer::install

  [30;46mWhich application preset do you want to install?[39;49m
  [[33m1[39m] Web
  [[33m2[39m] Cli
  [[33m3[39m] gRPC
  Make your selection [33m(1)[39m:
```

> **Note**
> When something went wrong during installation process, you can always restart it by running `composer install`
> command.

Once the application is installed, `README.md` file will be generated in the root directory of the project with the
instructions on how to start the application server and how to run the application.

> **Warning**
> RoadRunner application server will be downloaded automatically. `php-curl` and `php-zip` required.

## Running the Server

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

By default, the application will be available on `http://localhost:8080`.

> **Note**
> Read more about application server in the [Getting started — Application Server](../start/server.md) section.

<hr>

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Application server](../start/server.md)
* [Directory Structure](../start/structure.md)
* [First HTTP controller](../start/http-basics.md)
