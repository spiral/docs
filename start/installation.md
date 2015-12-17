# Installation and Requiments
Spiral Framework uses **Composer** to resolve dependencies and install required components. You can check how to install composer [here](https://getcomposer.org/download/).

## Server Requiments
Make sure that your server is configured with following PHP version and extensions:
* PHP 5.5+
* OpenSSL Extension
* MbString Extension
* Tokenizer Extension

## Installation
One of fastest ways to install a fresh spiral application is to use the composer command:

```
php composer.phar create-project spiral/application
```

Initial application installation will automatically start framework configuration using console command `configure`, this command will that all nesessary directories have the correct permissions and are available for application, also it will perform translator indexation and view compilation (you can register you own commands in configure sequence, see [**Console Dispatcher**](/console/commands.md)).

You can run application configuration at any moment, simply execute:

```
./spiral configure
```

Make sure that `spiral` has right execute permissions (`chmod +x spiral`).

> You can run comment with increaced verbosity to get more details.

# Enviroment settings
Default spiral application and framework core utilize [dotenv package](https://github.com/vlucas/phpdotenv) which provides you ability to enter your enviroment specific settings into `.env` file.

> Enviroment file `.env` will be automatically created and pre-configured as moment of application installation, however in a future you have to manage file creation manually (you can simply copy or rename `.env.sample` and later run `spiral app:key` to ensure that unique encryption key were set).
