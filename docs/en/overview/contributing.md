# Overview — Contribution Guide

Are you interested in helping out with the Spiral Framework? It's an open-source project that needs the help of
developers like you to keep it running and make it even better. There are lots of ways you can get involved.

## Pull Requests

One way to contribute to the framework is by submitting pull requests on GitHub.
If you have a fix or improvement, you can submit a [pull request](https://github.com/spiral/framework/pulls) and it will
be reviewed by the maintainers. There are a few requirements that you should be aware of when making a pull request:

### Declare strict types

The framework requires that all PHP files
declare [strict types](https://www.php.net/manual/en/language.types.declarations.php#language.types.declarations.strict)
using the `declare(strict_types=1);` directive at the top of the file. This helps ensure that code is type-safe and
reduces the risk of bugs and errors.

Here is an example of how this directive should be used:

```php
<?php

declare(strict_types=1);

namespace App;

// ...
```

### Follow the PSR-12 coding standard

Make sure your code follows the [PSR-12](https://www.php-fig.org/psr/psr-12/) coding standard. This is a set of
guidelines for how code should be formatted and structured, so that it's consistent and easy to read.

#### Check the code

```terminal
./vendor/bin/php-cs-fixer fix --config=.php-cs-fixer.dist.php -vvv --dry-run --using-cache=no
```

#### Automatically fix the code style

```terminal
./vendor/bin/php-cs-fixer fix --config=.php-cs-fixer.dist.php -vvv --using-cache=no
```

> **Note**
> The package will force `CL` endings.

### Include tests

Important thing to keep in mind when submitting code to the Spiral Framework is to include tests. This helps make sure
that the code you're submitting is working the way it should, and that it doesn't cause any new problems and, it makes
it easier for maintainers to verify that your code is correct.

#### Run tests

```terminal
./vendor/bin/phpunit 
```

### Use Psalm

Before you submit any code, you have to use [Psalm](https://psalm.dev/) to check for any mistakes or problems. This
makes sure that your code is correct and follows the best ways of doing things. The Spiral Framework uses Psalm's error
level, **"4"** when checking code. It's an easy step to make sure your contribution is up to the standard.

#### Run Psalm

```terminal
./vendor/bin/psalm --no-cache
```

### Keep it simple

One more thing to keep in mind when contributing to the Spiral Framework is to try and keep your code as simple and
straightforward as possible. The maintainers prefer solutions that are easy to understand and implement over ones that
are overly complex. So, when making a pull request, try to think of the simplest way to solve the problem at hand and
present it that way. It will help your pull request to get accepted faster.

## Support Questions

If you have any questions or need advice or suggestions, feel free to join our [Discord](https://discord.gg/TFeEmCs)
channel for support from the framework maintainers and community members.

## Issues

If you come across any issues or security vulnerabilities while using the Spiral Framework, please report them. The
maintainers take these matters very seriously and will do their best to address them as soon as possible. You can report
issues or vulnerabilities by opening an [issue](https://github.com/spiral/framework/issues) in the Spiral Framework's ]
GitHub repository.

## Commercial Support

It's important to note that the Spiral Framework and all related components are maintained
by [Spiral Scout](https://spiralscout.com/).

If you would like to support the development of the Spiral Framework, you can become
a [sponsor](https://github.com/sponsors/roadrunner-server).

Additionally, if you need commercial support, you can contact the Spiral Scout team
at [team@spiralscout.com](mailto:team@spiralscout.com). They will be happy to help you with any questions or issues you
may have.

## Licensing

Spiral Framework and its components will remain under [MIT license](/license.md) indefinitely.
