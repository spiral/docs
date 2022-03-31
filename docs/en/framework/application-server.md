# Framework - Application Server
The framework application server is based on [RoadRunner](https://roadrunner.dev) but includes multiple additions specific to Spiral such as
Queue, Scheduler, and GRPC integrations.

> Attention, you need some basic knowledge of Golang to customize the application server.

## Downloading Application Server
You can download the application server directly from [release page](https://github.com/spiral/framework/releases). 

If your PHP includes `php-cli` and `php-zip` extensions you can also let spiral to download server automatically:

```bash
$ ./vendor/bin/spiral get
```

## Running the Server
The spiral server is easy to run on the default `:8080` port:

```bash
$ ./spiral serve -v -d
```

You can observe the memory consumption of your workers in realtime and other information via

```bash
$ ./spiral http:workers -i
```

> Use similar commands `jobs:workers` and `grpc:workers` to check other dispatchers.

## Building Application Server
A lot of the sections in this documentation will explain how to extend your application capabilities by adding your own RoadRunner services, middleware, or data providers. It's essential to learn how to build a server on your own.

> You are not required to learn Golang or build the application server by yourself, the default build will cover all of the framework features.

#### Install Golang
To build an application server, you need [Golang 1.13+](https://golang.org/dl/) to be installed.

#### Create main.go
Download default [main.go](https://github.com/spiral/framework/blob/master/main.go) file, we are going to use it later to register custom services. You can store this file in the root of your project or other location.

#### Initiate go modules
Go Modules is a Golang approach to manage your application dependencies, and it's very similar to the Composer. You can initiate a blank 
`go.mod` file (`composer.json` analog) by running the following command in the same directory as your application.
 B
```bash
$ go mod init {repository-name}
``` 

If you are planning to install custom extensions, make sure that `{repository-name}` points to the repository Golang can download, for example
`github.com/username/my-application`.

#### Build the Server
You can build your application server by only running:

```bash
$ go build
```
