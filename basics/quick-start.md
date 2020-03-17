# Quick Start
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


### Lighter up

### Developer Mode

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
