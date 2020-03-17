# Quick Start [Work in Progress]
Spiral framework contains a lot of components build to operate seamlessly with each other.
In this article we will show how to create a demo blog application with REST API, ORM, Migrations, request validation, 
queue and user authorization.

> The components and approaches will be covered at basic levels only. Read the corresponding 
sections to gain more information.

## Installation
Use composer to install the default `spiral/app` bundle with most of components out of the box:

```bash
$ composer create-project spiral/app spiral-demo
$ cd spiral-demo
```

> Use the opposite build `spiral/app-cli` to install spiral with minimal dependencies.

If everything installed correctly you can open your application immediately:

```bash
$ ./spiral serve -v -d
```

You just started [application server](/framework/application-server.md). The same server can used on production making your
development environment similar to the final setup. Out of the box server includes instruments to write portable applications
with HTTP/2, GRPC, Queue, WebSockets, etc.

By default, the application will be available on `http://localhost:8080`. Build includes multiple pre-defined pages.

> Check the exception page `http://localhost:8080/exception.html`, at the right part of this page you can see all interceptors
> and middleware included in the default build. We will turn some of them off. 

## Configure
Spiral applications configured using ENV variables which can be used in config files. Default configuration is located
in `app/config` in a form of .PHP files. You can also tweak the application server and its plugins via `.rr.yaml` file.

The list of application dependencies located in `composer.json` and activated in a form of Bootloaders in `app/src/App.php`.

### Lighter Up
We won't need translation, session and cookies in our application. Remove this components and their bootloaders.

Delete following bootloaders from `app/src/App.php`:

```php
Framework\I18nBootloader::class,
Framework\Security\EncrypterBootloader::class,

// from http
Framework\Http\CookiesBootloader::class,
Framework\Http\SessionBootloader::class,
Framework\Http\CsrfBootloader::class,
Framework\Http\PaginationBootloader::class,

// from views 
Framework\Views\TranslatedCacheBootloader::class,

// from APP
Bootloader\LocaleSelectorBootloader::class,
```

You can delete the corresponding dependencies in `composer.json` as well:

```bash
"spiral/cookies": "^1.0",
"spiral/csrf": "^1.0",
"spiral/session": "^1.1",
"spiral/translator": "^1.2",
"spiral/encrypter": "^1.1",
```

Delete following files and directories as not longer required `app/locale`, `app/src/Bootloader/LocaleSelectorBootloader.php`, `app/src/Middleware`.

> Alternatively, start from `spiral/app-cli` build and add needed components on-demand.

### Developer Mode
To simplify the tweaking of application restart the application server in developer mode. In this mode the server use
only one worker and reloads it after every request.

```bash
$ .\spiral.exe serve -v -d -o "http.workers.pool.maxJobs=1" -o "http.workers.pool.numWorkers=1"
```

You can also create and use alternative configuration file via `-c` flag of `spiral` application.

> Note that the application is currently in non functional state (check the Exception message) as the `app/views/layout/base.dark.php` still
> refers to application locale. 

### HTTP Components

### Database Connection

### Connect Faker 

### Routing

### Annotated Routing

### Domain Core

## Scaffold Database

### Define ORM Entities

### Generate Migration

### Create Relations 

## Service

### Prototyping

### Persist Entity

## Console Command

## Controller

### Data Grid

## Validate Request

### Hydrate Entity

### Combine 

## Render Template

### Create Layout

### Implement View

## Background Task

### Create Endpoint

### Create Job Handler

### Push Background Task

## Authenticate User

### Connect the Component

### Authenticate User

### Attach Middleware
