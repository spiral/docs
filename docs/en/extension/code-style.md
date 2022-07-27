# Extensions - Code Style
Spiral framework ships with an extension intended to unify the coding style across its components and your applications. 
The extension code style is based on a strict [PSR-12](https://www.php-fig.org/psr/psr-12/) with no exceptions.

The extension repository: https://github.com/spiral/code-style

The actual formatting and style-checking is based on:
- [PHP_CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer/)
- [PHP CS Fixer](https://cs.symfony.com/)

## Installation
To install the extension:

```bash
composer require --dev spiral/code-style
```

## Check the code
To check code-style compliance: 

```
# vendor/bin/spiral-cs check <dir1> <dir2> <file1>....
vendor/bin/spiral-cs check src tests
```

## Automatically fix the code style
To automatically fix code-style:
```
# vendor/bin/spiral-cs fix <dir1> <dir2> <file1>....
$ vendor/bin/spiral-cs fix src tests
```

> Note, the extensions will force CL endings.
