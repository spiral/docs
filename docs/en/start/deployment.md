# Getting started — Deployment

Deploying a Spiral Framework application can be a complex task, as it requires various steps to be performed in order to
ensure that the application is properly configured and ready for production use. In this guide, we will go over
several strategies for deploying an application, including basic file transfer, using source control, build scripts,
and Docker builds.

Each strategy will include an example and step-by-step instructions to help you understand the process. It's important
to note that the specific steps may vary depending on your setup and requirements, so it is recommended to test the
deployment process on a development environment before deploying to a production environment.

Deploying requires several steps to be taken in order to ensure that the application is properly configured,
functional and ready for production use. In this section, we will go over the typical steps:

1. **Uploading the application code:** The code for the application is transferred from the development environment to
   the production server. This can be done using various methods such as Git.

2. **Installing dependencies:** The application's dependencies, such as external libraries and modules, need to be
   installed. This is typically done through the Composer package manager.

3. **Running database migrations:** If there have been any changes to the database schema, the necessary migrations need
   to be run to update the production database.

4. **Clearing (and optionally, warming up) the cache:** The application's cache should be cleared to ensure that any
   stale data is removed.

5. **Testing the application:** The application should be tested to ensure that it is functioning correctly on the
   production server.

6. **Deploying the application:** The application is made live and made available to users. This may include adjusting
   the server's settings, configuring a load balancer, or other tasks as necessary.

7. **Monitoring the application:** The application's performance and usage should be monitored to ensure that it is
   running smoothly and to detect and troubleshoot any issues that may arise.

Let's explore several deployment strategies that can be used to deploy an application. Each strategy has its own
advantages and disadvantages and is suited to different types of projects and environments. By understanding these
strategies and the steps involved in each, you can choose the best approach for your specific situation.

## Deployer

Using automation tools like [Deployer](https://deployer.org/) can automate the deployment process and handle tasks such
as database migrations, cache clearing and more. They also allow for faster deployment and rollback processes, and can
be integrated with other tools like load balancers and monitoring systems. This can make the deployment process more
efficient and less prone to errors, and makes it easier to keep track of which version of the code is currently on the
production server, making it hard to rollback in case of errors.

Deployer is a popular deployment tool that can be used to automate the deployment process for a Spiral Framework
application. One of the advantages of using Deployer is that it can provide "zero downtime" deployment.

Zero downtime deployment is a technique that allows you to update your application without interrupting the service to
your users. This is achieved by deploying the new version of the application alongside the old version, and then
swapping them out once the new version is ready. This allows for a seamless transition, as users are not affected by any
downtime during the deployment process.

Deployer provides the capability of Zero Downtime deployment by creating a new release of the application in a different
directory and switching the symlink to the new release.

### Installing

To install Deployer, run next command in your project dir:

```terminal
composer require --dev deployer/deployer
```

### Configuration

To initialize deployer in your project run:

```terminal
./vendor/bin/dep init
```

Deployer will ask you a few questions and after finishing you will have a `deploy.php` or `deploy.yaml` file. This is
your deployment recipe. It contains hosts, tasks and requires other recipes.

Here is an example of a `deploy.php` file:

```php deploy.php
namespace Deployer;

require 'recipe/spiral.php';

set('repository', 'https://github.com/xxx/my-app');

add('shared_files', []);
add('shared_dirs', []);
add('writable_dirs', []);

host('example.org')
    ->set('remote_user', 'deployer')
    ->set('deploy_path', '/var/www/my-app');

after('deploy:failed', 'deploy:unlock');

desc('Deploys your project');
task('deploy', [
    'deploy:prepare',
    'deploy:environment',
    'deploy:vendors',
    'spiral:encrypt-key',
    'spiral:configure',
    'deploy:download-rr',
    'deploy:publish',
    'deploy:restart-rr'
]);
```

To connect to the remote host we need to specify an identity key or private key. We can add our identity key directly
into the host definition, but it's better to put it in the `~/.ssh/config` file:

```bash ~/.ssh/config
Host example.org
  IdentityFile ~/.ssh/id_rsa
```

Now let's provision our server. As our host doesn't have user deployer, we are going to override `remote_user` for
provision via `-o remote_user=root`.

```terminal
dep provision -o remote_user=root
```

Deployer will ask you a few questions during provisioning: php version, database type, etc. Next Deployer will configure
our server and create the deployer user. Provision takes around 5 minutes and will install everything we need to run a
website. A new website will be configured at `deploy_path`.

### Deploying

To deploy your application, run the following command:

```terminal
./vendor/bin/dep deploy
```

> **Note**
> If deploy failed, Deployer will print an error message and which command was unsuccessful. Most likely we need to
> configure the correct database credentials in `.env` file or similar.

#### CI/CD

Deployer can be used in CI/CD pipelines. Read more about it in [Deployer CI/CD](https://deployer.org/docs/7.x/ci-cd)
section.

<hr>

## Docker

Docker allows for easy deployment, scaling, and management of application containers. It also makes it easy to replicate
the production environment and test applications locally, which makes it easier to find and fix issues.

This method involves building a Docker image of the application, and then running the image on the production server.

### Example

1. Create a new `Dockerfile`
2. Add the following code to Dockerfile

```dockerfile 
# Build rr binary
FROM spiralscout/roadrunner:latest as rr

# Clone the project
FROM alpine/git as git

ARG REPOSITORY=https://github.com/xxx/my-app
ARG BRANCH=master
RUN git clone -b $BRANCH $REPOSITORY /app

WORKDIR /app/bin

# Configure PHP project
FROM php:8.1.3-cli-alpine3.15 as backend

RUN apk add --no-cache \
        curl libcurl wget libzip-dev libmcrypt-dev libxslt-dev libxml2-dev icu-dev zip

RUN docker-php-ext-install \
        opcache zip xsl dom exif intl pcntl bcmath sockets

COPY --from=rr /usr/bin/rr /app

ARG APP_VERSION=v1.0
ENV COMPOSER_ALLOW_SUPERUSER=1

WORKDIR /app

RUN curl -s https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin/ --filename=composer

RUN composer config --no-plugins allow-plugins.spiral/composer-publish-plugin false
RUN composer install --optimize-autoloader --no-dev

RUN docker-php-source delete && apk del ${BUILD_DEPENDS}

EXPOSE 8080/tcp

CMD ./rr serve -c .rr.yaml
```

3. Run `docker build -t myapp .` to build the image
4. Run `docker run -p 8080:8080 myapp` to start the container

<hr>

## Source Control

A popular way to deploy an application is by using source control. This involves connecting your production server to a
remote repository, such as Git or SVN, and then deploying the application by pulling in the latest changes from the
repository.

### Example

1. Connect to the production server via SSH
2. Navigate to the root directory of your application on the server
3. Run the command `git pull origin master` to pull in the latest changes from the remote repository
4. Once the changes are pulled in, run `composer install --optimize-autoloader --no-dev` to install the dependencies
5. Run database migrations and other necessary console commands
6. Restart the RoadRunner server workers `./rr reset`

> **Warning:**
> Using this strategy without automation can make the deployment process more time-consuming and prone to errors. Every
> time a deployment is needed, developers would need to manually pull the code from the repository and run any necessary
> console commands in a specific order. This increases the risk of human error, such as running the commands in the
> wrong order or forgetting to run a command, which can lead to issues with the application.

<hr>

## File Transfer

One of the simplest ways to deploy a Spiral Framework application is through basic file transfer. This can be done using
FTP, or SCP. The process involves uploading the application files from your local machine to the production server.

### Example

1. Connect to the production server using FTP client (FileZilla, WinSCP)
2. Navigate to the root directory of your application on the server
3. Select all of the application files on your local machine and upload them to the server
4. Once the upload is complete, run `composer install --optimize-autoloader --no-dev` to install the dependencies
5. Run database migrations and other necessary console commands
6. Restart the RoadRunner server workers `./rr reset`

> **Warning:**
> This strategy is considered less efficient and less secure than other methods. It can take a long time to upload all
> the files manually, and there is a risk of data loss or corruption if the transfer is interrupted. Additionally, it
> can be difficult to keep track of which version of the code is currently on the production server, making it hard to
> rollback in case of errors.