# Distribution

The `spiral/distribution` component is responsible for providing public HTTP links on arbitrary resources. In most
cases, this will be the same address as the address of the site itself, however, in some cases, resources may be located
on external servers such as [Amazon CloudFront](https://aws.amazon.com/cloudfront/) or some other CDN. In these cases,
generating a public link to the resource needs to use a specific API of the provider, or write one's own code for the used
CDN. The component makes this interaction easier and provides a number of built-in drivers for generating URIs to
external suppliers.

## Installation

Use Composer to install the component:

```bash
composer require spiral/distribution
```

### Framework Integration

> **Note**
> Please note that the spiral/framework >= 2.8 already includes this component.

To enable the component, you just need to add the `Spiral\Distribution\Bootloader\DistributionBootloader` class to the 
bootloader list, which is located in the class of your application.

```php
protected const LOAD = [
    // ...
    // Added distribution bootloader
    \Spiral\Distribution\Bootloader\DistributionBootloader::class,
    // ...
];
```

## Configuration

The configuration file for this component looks like the one below. Just create a `distribution.php` file and add it to the 
directory with the rest of your configuration files (e.g. `~/app/config/distribution.php`).

```php
<?php

return [

    /**
     * -------------------------------------------------------------------------
     *  Default Distribution Resolver Name
     * -------------------------------------------------------------------------
     *
     * Here you can specify which of the resolvers you want to use in the
     * default for all work with URI generation. Of course, you can use
     * multiple resolvers at the same time using the distribution library.
     *
     */

    'default' => env('DISTRIBUTION_RESOLVER', 'local'),

    /**
     * -------------------------------------------------------------------------
     *  Distribution Resolvers
     * -------------------------------------------------------------------------
     *
     * Here is each of the resolvers configured for your application.
     * Of course, the examples of customizing each available distribution supported
     * by Spiral are shown below to simplify development.
     *
     */

    'resolvers' => [
        'local' => [
            'type' => 'static',
            'uri'  => env('APP_URL', 'http://localhost')
        ],

        'cloudfront' => [
            'type' => 'cloudfront',
            'key' => env('AWS_CF_KEY'),
            'domain' => env('AWS_CF_KEY'),
            'private' => env('AWS_CF_PRIVATE_KEY'),
        ],

        's3' => [
            'type' => 's3',
            'region' => env('S3_REGION'),
            'bucket' => env('S3_BUCKET'),
            'key' => env('S3_KEY'),
            'secret' => env('S3_SECRET'),
        ],
    ],

];
```

> **Note**
> The configuration above is only available when used with a Spiral Framework.

### Manual Configuration (Outside The Framework)

This way of using the component is required only if it is installed separately, outside the framework.

First you need to create a manager instance where all your uri resolvers will be stored. After that, you can add and 
get arbitrary resolvers from it by the desired name.

```php
<?php

$manager = new \Spiral\Distribution\Manager();

$manager->add('resolver-name', new CustomResolver());

$manager->resolver('resolver-name'); // object(CustomResolver)
```

After that, you can add there either your own managers, or provided by the component, such as for example "static".

```php
<?php

use Nyholm\Psr7\Uri;
use Spiral\Distribution\Manager;
use Spiral\Distribution\Resolver\StaticResolver;

$manager = new Manager();
$manager->add('local', new StaticResolver(new Uri('https://static.example.com')));
```

## Usage

Once you've configured your component, you can start using it.

If you are using the Spiral Framework, the manager is already configured. You can get it from 
[the container](/framework/container.md) or via [dependency injection](/framework/container.md#dependency-injection).

```php
<?php

use Spiral\Distribution\DistributionInterface;

class FilesController
{
    public function showImage(DistributionInterface $dist): string
    {
        $resolver = $dist->resolver('local');

        return (string)$resolver->resolve('example/image.jpg');
    }
}
```

If you need a default resolver defined in the "default" configuration section, you do not need to 
get the entire manager instance. You can get the resolver you want from the container right away.

```php
<?php

use Spiral\Distribution\UriResolverInterface;

class FilesController
{
    public function showImage(UriResolverInterface $resolver): string
    {
        return (string)$resolver->resolve('example/image.jpg');
    }
}
```

You may have noticed that after getting the resolver in the examples above, the `resolve()` method is used with a 
relative path to the file. It takes a string value as an argument and returns the implementation of
[the PSR-7 `Psr\Http\Message\UriInterface`](https://www.php-fig.org/psr/psr-7/).

```php
$uri = $resolver->resolve('path/to/file.txt');
//
// Expected:
//  object(Psr\Http\Message\UriInterface)
//
```

> **Note**
> Some resolvers support additional options when getting a link, 
> for example: `$cloudfront->resolve('path/to/file.txt', expiration: new \DateInterval('PT60S'));`

### Static URI Resolver

This type of a resolver generates an address to a resource simply by adding the passed file link to the end of the URI 
specified in the resolver configuration.

To configure this type of resolver, you only need to specify two required fields.

```php
return [
    // ...
    'resolvers' => [
        // ...
        'local' => [
            //
            // Required key of resolver type.
            // For static resolver, it must contain the "static" string value.
            //
            'type' => 'static',

            //
            // Required key of static server url.
            //
            'uri'  => env('APP_URL', 'http://localhost')
        ],
    ]
];
```

Unlike a similar method used to generate an address for a page in the [url generator](/http/routing.md#url-generation) 
router component, links can be arbitrary and configured on a separate server designed to serve static content.

In this way, if you pass an arbitrary file string to the `resolve()` method, you will receive a physical http link to this 
file. If the base uri is defined as "`http://localhost`", the result will be as follows:

```php
/** @var \Spiral\Distribution\Resolver\StaticResolver $resolver */
$resolver = $manager->resolver('local');

echo $resolver->resolve('path/to/file.txt');
//
// Expected:
//  string(33) "http://localhost/path/to/file.txt"
//
```

### CloudFront URI Resolver

CloudFront is a popular static distribution service used in conjunction with Amazon services. To use it, you must 
install the `aws/aws-sdk-php` package using the Composer.

```bash
composer require aws/aws-sdk-php ^3.0
```

After registering and creating your statics server in the AWS 
services, [you will receive](https://console.aws.amazon.com/cloudfront/home) the parameters for setting. In addition, 
you will need a "private key file" and "access key id", which you can find on the "CloudFront key pairs" tab 
on "[Security Credentials](https://console.aws.amazon.com/iam/home#/security_credentials)" page.

To configure this resolver, simply specify the connection parameters in the configuration sections:

```php
return [
    // ...
    'resolvers' => [
        // ...
        'cloudfront' => [
            //
            // Required key of resolver type.
            // For CloudFront, it must contain the "cloudfront" string value.
            //
            'type' => 'cloudfront',

            //
            // Required key of CloudFront access key id.
            // This must contain string value like "AAAABBBBCCCCDDDDEEEE".
            //
            // Identifier can be found on your personal "security credentials" page here:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'key' => env('AWS_CF_KEY'),
            
            //
            // Required key of CloudFront private key.
            // This must be a private key string value or a path to a private key file.
            //
            // Identifier can be also found on "Security Credentials" page here:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            // Please note that you can download the private key file only
            // during its generation!
            //
            'private' => env('AWS_CF_PRIVATE_KEY'),

            //
            // Required key of CloudFront domain name.
            // This must contain string value like "example.cloudfront.net".
            //
            // Domain can be found on "CloudFront Distributions" page here:
            //  - https://console.aws.amazon.com/cloudfront/home
            //
            'domain' => env('AWS_CF_DOMAIN'),
            
            //
            // Optional key of CloudFront file prefixes.
            // This must contain string like "path/to/directory". In this case,
            // this prefix will be added for each file when generating url.
            //
            'prefix' => env('AWS_CF_PREFIX'),
        ],
    ]
];
```

If you decide to create a resolver yourself, you can use the same settings passed to the constructor of 
the resolver used to work with the CloudFront service.

```php
//
// The use of PHP 8 named arguments in the constructor is for clarity
//
$cloudfront = new \Spiral\Distribution\Resolver\CloudFrontResolver(
    keyPairId: 'AAAABBBBCCCCDDDDEEEE',
    privateKey: \file_get_contents(__DIR__ . '/path/to/key.pem'),
    domain: 'example.cloudfront.net',
    prefix: 'path/to/files'
);

$url = $cloudfront->resolve(...);
```

The CloudFront resolver receives as the first argument of the `resolve()` method a link to a file for which a public 
address should be generated and, as the second, optional, the lifetime (expiration) of this link.

The expiration time can be specified in several formats. It can be:

- PHP builtin [`\DateInterval` object](https://www.php.net/manual/en/class.dateinterval.php).
- Instance of [`\DateTimeInterface` interface](https://www.php.net/manual/en/class.datetimeinterface.php).
  In this case, the expiration interval is counted from the moment the link is generated.
- String value in [PHP time duration format](https://www.php.net/manual/en/dateinterval.construct.php).
- Integer value in seconds.

Below are examples of each of the valid formats:

```php
$file = 'path/to/file.txt';

// DateInterval object
$url = $cloudfront->resolve($file, new DateInterval('PT30S'));

// Instance of DateTimeInterface
$url = $cloudfront->resolve($file, new DateTime('+30 sec'));

// Duration in string format
$url = $cloudfront->resolve($file, 'PT30S');

// Duration in int format
$url = $cloudfront->resolve($file, 30);
```

In case of any special circumstances, you can replace the current time generator and expiration parser. In addition, 
you can also set a default value for all generated links within a given resolver.

```php
$cloudfront = (new \Spiral\Distribution\Resolver\CloudFrontResolver(...))
    //
    // With custom "current time" generator.
    //
    // The time generator must be an implementation of the
    // \Spiral\Distribution\Internal\DateTimeFactoryInterface interface.
    //
    ->withDateTimeFactory(new CustomCurrentDateGenerator())

    //
    // With custom "expiration time" parser.
    //
    // The "expiration time" format parser must be an implementation of the
    // \Spiral\Distribution\Internal\DateTimeIntervalFactoryInterface interface.
    //
    ->withDateTimeIntervalFactory(new CustomExpirationParser())

    //
    // With default "expiration time" value.
    //
    // The value must be correct for the time specified in the
    // parser of given URI resolver.
    //
    ->withExpirationDate('PT30S');
```

### S3 URI Resolver

If, for some reason, you cannot use the CloudFront resolver (for example, in the case of using 
a [Minio Server](https://docs.min.io/)), you can use the resolver that generates links to the S3 server. To use it, you
must also install the `aws/aws-sdk-php` package using the Composer.

```bash
composer require aws/aws-sdk-php ^3.0
```

To use it with AWS S3, you need account credentials, and a working bucket which you can 
create [on "Amazon S3" page](https://s3.console.aws.amazon.com/s3/home). After creating the bucket, you will need to 
fill in the following configuration parameters.

```php
return [
    // ...
    'resolvers' => [
        // ...
        's3' => [
            //
            // Required key of resolver type.
            // For S3, it must contain the "s3" string value.
            //
            'type' => 's3',

            //
            // Required string key of S3 region like "eu-north-1".
            //
            // Region can be found on "Amazon S3" page here:
            //  - https://s3.console.aws.amazon.com/s3/home
            //
            'region' => env('S3_REGION'),

            //
            // Optional key of S3 API version.
            //
            'version' => env('S3_VERSION', 'latest'),

            //
            // Required key of S3 bucket.
            //
            // Bucket name can be found on "Amazon S3" page here:
            //  - https://s3.console.aws.amazon.com/s3/home
            //
            'bucket' => env('S3_BUCKET'),

            //
            // Required key of S3 credentials key like "AAAABBBBCCCCDDDDEEEE".
            //
            // Credentials key can be found on "Security Credentials" page here:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'key' => env('S3_KEY'),

            //
            // Required key of S3 credentials private key.
            // This must be a private key string value or a path to a private key file.
            //
            // Identifier can be also found on "Security Credentials" page here:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'secret' => env('S3_SECRET'),

            //
            // Optional key of S3 credentials token.
            //
            'token' => env('S3_TOKEN', null),

            //
            // Optional key of S3 credentials expiration time.
            //
            'expires' => env('S3_EXPIRES', null),

            //
            // Optional key of S3 API endpoint URI.
            //
            'endpoint' => env('S3_ENDPOINT', null),
            
            //
            // Optional key of S3 API file prefixes.
            // This must contain string like "path/to/directory".
            //
            // In this case, this prefix will be added for each file when
            // generating url.
            //
            'prefix' => env('S3_PREFIX'),

            //
            // Optional additional S3 options.
            // For example, option "use_path_style_endpoint" is required to work
            // with a Minio S3 Server.
            //
            // Note: This "options" section is available since framework >= 2.8.6
            //
            'options' => [
                'use_path_style_endpoint' => true,
            ]
        ],
    ]
];
```

If you decide to create a resolver yourself, you can use the same settings passed to the constructor of 
the resolver used to work with the S3 service.

```php
//
// The use of PHP 8 named arguments in the constructor is for clarity
//
$s3 = new \Spiral\Distribution\Resolver\S3SignedResolver(
    client: new \Aws\S3\S3Client([
        'version' => 'latest',
        'region'  => 'eu-north-1',
        'credentials' => new \Aws\Credentials\Credentials(
            key: 'key',
            secret: file_get_contents(__DIR__ . '/path/to/secret.pem')
        )
    ]),
    bucket: 'bucket-name',
    prefix: 'path/to/files'
);

$url = $s3->resolve(...);
```

After registering a resolver, you will be able to create a URI to a file using the `resolve()` method. By analogy with 
the CloudFront implementation, you can also pass a second `expiration` argument to this method, which means the lifetime 
of the generated URI.

```php
$url = $s3->resolve($file, new DateTime('+30 sec'));
```

All similar methods for specifying the global URI expiration, the "current time" generator, and the "expiration time" 
parsers are also available.

### Custom URI Resolver

In some cases, you may find tasks for generating URI's that do not fit the existing implementations of resolvers. In 
this case, you can register your own resolver class in the config. To pass additional arguments to the constructor of 
this resolver, simply specify the `options` section in the configuration file.

```php
return [
    // ...
    'resolvers' => [
        // ...
        'custom' => [
            //
            // Required key of resolver class. This is class must implement
            // \Spiral\Distribution\UriResolverInterface interface.
            //
            'type' => \Example\CustomResolver::class,

            //
            // Optional "options" array section.
            //
            'options' => [
                // list of constructor arguments...
            ],
        ],
    ]
];
```

In some cases, this registration method may not work for you. If any dependencies from the container are 
required in the parameters of the constructor, you should use [the bootloader](/framework/bootloaders.md).
