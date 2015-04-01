# Installation and Requirements
Spiral Framework uses `Composer` to resolve dependencies and core file. Check how to install composer 
[here] (https://getcomposer.org/download/).

## Requirements
Spiral Framework has following server requirements:

* PHP 5.4+
* PHP MbString
* PHP OpenSSL

## Installation
To install fresh spiral application use following command:
```
php composer.phar create-project spiral/application directory dev-master
```

Spiral will configure all folders and permissions automatically after installation. If you want to 
run configure command later:

```
>./spiral.cli core:configure
Verifying runtime directory existence and permissions.
Runtime directory permissions updated.

Installing all available modules.
Module 'spiral/profiler' successfully installed/updated.

Re-indexing available console commands.
Console commands re-indexed, 24 commands found.

Creating initial i18n bundles cache.
Scanning i18n function usages...
Scanning Localizable classes...
Strings found: 27 in 8 bundle(s).

View cache successfully generated.

Application configured.
```

## Environment
By default spiral will check what environment to use based on content of `runtime/environment.php` file
(this behaviour can be changed by custom Application class). Right after installation, environment value
will be set as `development`, if you want to change it execute following command:

```
>./spiral.cli core:environment production -v
Environment set to 'production'.
Following configuration files will be altered by this environment:
./debug.php
./views.php
```

> Environment change will alter applicationID, which will automatically invalidate configuration
and view cache. Read more about configuring spiral application in Core / Configuration section.