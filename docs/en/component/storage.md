# Storage and Cloud distribution

The component `spiral/storage` provides powerful storage abstraction thanks to the
wonderful [Flysystem](https://github.com/thephpleague/flysystem) PHP package by Frank de Jonge. The Storage component
integration provides simple drivers for working with local filesystems and Amazon S3. Even better, it's super simple
to switch between these storage options between your local development machine and production server as the API remains
the same for each system.

Please note that, unlike classical file systems, the store component provides an API which provides the operations of
writing a file, checking its existence, reading and getting a public address of this file. All operations (which
classic file systems have) for working with directories, or a list of files are not available.

To install the component:

```terminal
composer require spiral/storage
```

## Framework Integration

> Please note that the spiral/framework >= 2.8 already includes this component.

Make sure to add `Spiral\Storage\Bootloader\StorageBootloader` to your App class:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Storage\Bootloader\StorageBootloader::class,
    // ...
];
```

## Configuration

A storage config file should be located (by default) at `app/config/storage.php`. Within this file, you may configure
the
servers (the "`servers`" section) and the specific storages (the "`buckets`" section) that your servers will use and
which you will use.

For example, the simplest configuration with one server and two storages might look like this:

```php
return [

    /**
     * -------------------------------------------------------------------------
     *  Default Storage Bucket Name
     * -------------------------------------------------------------------------
     *
     * Here you can specify which of the buckets you want to use by default for
     * all work with storages.
     *
     */

    'default' => env('STORAGE_SERVER', 'uploads'),

    /**
     * -------------------------------------------------------------------------
     *  Storage Servers
     * -------------------------------------------------------------------------
     *
     * Here is each of the servers configured for your application. Of
     * course, the examples of customizing each available server supported by
     * Spiral are shown below to simplify development.
     *
     */

    'servers' => [
        'static' => [
            'adapter' => 'local',
            'directory' => __DIR__ . '/../../runtime/static',
        ],

        's3' => [
            'adapter' => 's3', // or "s3-async"
            'region' => env('S3_REGION'),
            'bucket' => env('S3_BUCKET'),
            'key' => env('S3_KEY'),
            'secret' => env('S3_SECRET'),
        ],
    ],

    /**
     * -------------------------------------------------------------------------
     *  Storage Buckets
     * -------------------------------------------------------------------------
     *
     * Here is a list of specific buckets (or storages) that use the
     * server settings above. Each "server" section in this list must refer to a
     * valid server name in the list above.
     *
     * The list of settings in this case is also an example of use. You can
     * freely change the number of buckets and the type of settings as you wish.
     *
     */

    'buckets' => [
        'uploads' => [
            'server' => 'static',
            'prefix' => 'upload',
        ],

        'images' => [
            'server' => 'static',
            'prefix' => 'img',
        ],

        'videos' => [
            'server' => 's3',
        ],
    ],
];
```

> **Note**
> Please note that this configuration is only available when used with the Spiral Framework.

### Manual Configuration (Outside The Framework)

This way of using the component is required only if it is installed separately, outside the framework.

First you need to create a storage instance where all your buckets will be stored. After that, you can add and get
an arbitrary bucket from it by the desired name.

```php
$storage = new \Spiral\Storage\Storage();

$storage->add('example', \Spiral\Storage\Bucket::fromAdapter(
    new \League\Flysystem\Local\LocalFilesystemAdapter(__DIR__ . '/path/to/directory')
));

$file = $storage->bucket('example')
    ->write('file.txt', 'content');
```

As you may have noticed, you can use the
existing [flysystem adapters](https://flysystem.thephpleague.com/v2/docs/adapter/local/)
to create a bucket. Just install the one you want and add it to the store using the `Bucket::fromAdapter()` method.

### Local Server

The local server, as the name implies, is located in the local file system (in the same place where the executable code
of your application is located).

You have already seen an example of local server settings earlier, however, to simplify them, some optional sections
have been specially removed. Let's now take a look at the complete configuration of this type of server, leaving all
the possible configuration sections.

```php
return [
    'servers' => [
        'local' => [
            //
            // Server type name. For a local server, this value must be
            // a string value "local".
            //
            'adapter' => 'local',

            //
            // The required path to the local directory where your files will
            // be stored.
            //
            'directory'  => '/example/directory',

            //
            // Visibility mapping. Here you can set the default visibility for
            // files and the permissions for files and directories corresponding
            // to a certain type of visibility.
            //
            // The visibility value can only be "private" or "public".
            //
            'visibility' => [
                'public'  => ['file' => 0644, 'dir' => 0755],
                'private' => ['file' => 0600, 'dir' => 0700],

                'default' => 'public',
            ],
        ],
    ],

    'buckets' => [
        'bucket' => [
            //
            // Relation to an existing local server. Note that all further
            // bucket options are applicable only for this (i.e. local) server
            // type.
            //
            'server' => 'local',

            //
            // For buckets that use local servers, you can add a directory
            // prefix. In this case, the real physical path to the file will
            // look like: "/example/directory/prefix", where
            // the "/example/directory" string is the physical directory
            // specified in the server, and "prefix" - prefix for file
            // directories specified in the bucket.
            //
            'prefix' => 'prefix',
        ]
    ]
];
```

### S3 Server

This type of server is designed to interact with an external distributed file system using the S3 protocol. S3 is the
main protocol for communicating with [Amazon](https://s3.console.aws.amazon.com/s3/home) servers. In addition to Amazon
S3 itself, there are free alternatives that you can install and use on your own server, for
example [Minio Server](https://docs.min.io/).

To set up this type of server, you will need an existing bucket and personal authentication data. In addition, for
completeness, the example below will include parameters that are optional and have default values.

Please note that in order to interact with this type of servers, you must have one of the two available packages
installed (either). You must install the `league/flysystem-aws-s3-v3` or `league/flysystem-async-aws-s3` package using
the Composer.

```terminal
composer require league/flysystem-aws-s3-v3 ^2.0
// OR
composer require league/flysystem-async-aws-s3 ^2.0
```

During configuration, you should specify which of the packages you will use in the server's "adapter" section.
The `"s3"` value corresponds to the `league/flysystem-aws-s3-v3` package, while the `"s3-async"` value corresponds
to the `league/flysystem-async-aws-s3` package.

```php
return [
    'servers' => [
        'local' => [
            //
            // Server type name. For a S3 server, this value must be a string
            // value "s3" or "s3-async".
            //
            'adapter' => 's3',

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
            // Optional key of S3 files visibility. Visibility is "public"
            // by default.
            //
            'visibility' => env('S3_VISIBILITY', 'public'),

            //
            // For buckets that use S3 servers, you can add a directory
            // prefix.
            //
            'prefix' => '',

            //
            // Optional key of S3 API endpoint URI. This value is required when
            // using a server other than Amazon.
            //
            'endpoint' => env('S3_ENDPOINT', null),

            //
            // Optional additional S3 options.
            // For example, option "use_path_style_endpoint" is required to work
            // with a Minio S3 Server.
            //
            // Note: This "options" section is available since framework >= 2.8.5
            // See also https://github.com/spiral/framework/issues/416
            //
            'options' => [
                'use_path_style_endpoint' => true,
            ]
        ],
    ],

    'buckets' => [
        'bucket' => [
            //
            // Relation to an existing S3 server. Note that all further bucket
            // options are applicable only for this (i.e. s3 or s3-async) server
            // type.
            //
            'server' => 's3',

            //
            // The visibility value for a specific bucket type. Although you can
            // specify server-wide visibility, you can also override this value
            // for a specific bucket type.
            //
            'visibility' => env('S3_VISIBILITY', 'public'),

            //
            // In case you want to use another bucket using the main
            // server settings, you can redefine it by specifying the
            // appropriate configuration key.
            //
            'bucket' => env('S3_BUCKET', null),

            //
            // If the new bucket is in a different region, you can also
            // override this value.
            //
            'region' => env('S3_REGION', null),

            //
            // A similar thing can be done with the directory prefix in cases
            // where a particular bucket must refer to some other root directory.
            //
            'prefix' => 'custom-directory',
        ]
    ]
];
```

### Custom Server

In some cases, standard adapters may not be enough and in this case you may need to specify your own. You can also use
your config file to configure your custom adapter.

In this case, the adapter section must refer to the `League\Flysystem\FilesystemAdapter` implementation, and the
"options" section will contain an array of arguments passed to the constructor of this adapter.

```php
return [
    'servers' => [
        'custom' => [
            //
            // Server type name that contains class name of adapter.
            //
            'adapter' => \Custom\FlysystemAdapter::Class,

            //
            // Adapter's constructor arguments.
            //
            'options' => [
                // ...
            ]
        ],
    ],

    'buckets' => [
        'bucket' => [
            //
            // Relation to custom server.
            //
            'server' => 'custom'
        ]
    ]
];
```

## Usage

Finally, after we have familiarized ourselves with what types of servers the component supports, we can move on to
using them.

The storage architecture assumes 3 different levels of access to identical operations: Storage, Bucket and File. At
each of these levels, you can operate on files, but the differences are in how much data you have to transfer to a
particular method. At the highest "Storage" level you have to pass information about the bucket and the file. At the
"Bucket" level you pass info only about the file. Finally, at the "File" level you will not have to pass any additional
information.

In practice, it will look like this. Let's try to create a file `example.txt` in 3 different ways inside some controller
of our application.

```php
use Spiral\Storage\StorageInterface;

class UploadController
{
    public function createFile(StorageInterface $storage): array
    {
        $result = [];
        
        // 1. Storage level
        $result[] = $storage->create('bucket://example.txt');

        // 2. Bucket level
        $result[] = $storage->bucket('bucket')
            ->create('example.txt');

        // 3. File level
        $result[] = $storage->bucket('bucket')
            ->file('example.txt')
            ->create();

        return $result;
    }
}
```

The differences between the methods are as follows:

- When working with a storage, you should pass a file name in the URI-like format `[BUCKET_NAME]://[FILE_NAME]`.
  In all methods of working with a storage, this is the first string argument.

- In all methods of working with a bucket, this is a file name in an arbitrary format. Please also note that the leading
  slash does not affect the file location in any way and the names `file.txt` and `/file.txt` will be completely
  identical.

- In all methods of working with a specific file, no additional arguments are required.

Each of the methods has both its advantages and disadvantages. Just use what you like best.

Since in the example above we are using a controller, then at the same time we will use an alternative dependency,
which is suitable in cases of simple file saving, when many implementations of buckets are not required.

In such cases as below, the bucket used in the system by default will be selected. You can define the bucket in
the `'default'` section of your configuration.

```php
use Spiral\Storage\BucketInterface;
use Psr\Http\Message\ServerRequestInterface;

class UploadingController
{
    public function upload(ServerRequestInterface $request, BucketInterface $bucket): string
    {
        /** @var \Psr\Http\Message\UploadedFileInterface $file */
        foreach ($request->getUploadedFiles() as $i => $file) {
            $bucket->write("file-{$i}.txt", $file->getStream());
        }

        return \count($request->getUploadedFiles()) . ' files uploaded';
    }
}
```

From each of the levels, you can refer to the child.

**From The Storage:**

- `$storage->bucket('[bucket-name]'): BucketInterface`
- `$storage->file('[bucket-name]://[file-name]'): FileInterface`

**From The Bucket:**

- `$bucket->file('[file-name]'): FileInterface`

After we have got to know the options to use the same methods at different levels of the store, we
can go on to describe the available possibilities. Let's start!

### Create And Write

If you want to create a file, you can use one of the two available methods: `create()` or `write()`. The first creates
an
empty file where it is not created, and the second one allows you to additionally write arbitrary `string`
or `resource` stream content there.

For example, the code that creates a file might look like this.

```php
// Creating file from bucket
$file = $bucket->create('file.txt');

// Creating file from bucket with string content
$file = $bucket->write('file.txt', 'message');

// Creating file from bucket with resource stream content
$file = $bucket->write('file.txt', fopen(__DIR__ . '/local/file.txt', 'rb+'));
```

### Copy And Move

To copy files, use the `copy()` method, which contains one required argument with the name of a new file and one
optional - the bucket where the file should be copied. If the second argument is not specified, the compilation
bucket will be identical to the original one.

The `move()` method is completely similar to using the `copy()` method, but instead of copying it moves the file.

```php
$backup = $firstBucket->copy('from.txt', 'backup.txt');

$moved  = $firstBucket->move('backup.txt', 'to.txt', $secondBucket);
```

> Please note that when a file is moved (or copied) from a bucket with private permissions to a distribution with public
> permissions, the visibility of the file will also change to the one corresponding to this bucket.

### Delete

To delete a file, just use the `delete()` method. This method accepts an optional boolean argument which means deleting
an
empty directory where the file was located.

```php
$bucket->delete('file.txt');
```

### Reading Content

There are two ways to read the contents of an existing file: Using the `getContents()` and `getStream()` methods. The
first one returns the string content of the file, and the second resource is a stream for working with streaming data.

```php
$string = $bucket->getContents('text.txt');

$resource = $bucket->getStream('music.mp3');
```

### Existence Check

To check the existence of a file, use the `exists()` method, which returns a boolean value `true` if the file exists
in the bucket, or `false` if it does not exist.

```php
$isExists = $bucket->exists('file.txt');
```

### File Size

To get information about the size of a file, use the `getSize()` method, which returns the size of an existing file in
bytes.

```php
$bytes = $bucket->getSize('file.txt');
```

### Last Modification Time

To retrieve information about the date of the last modification of a file, use the `getLastModified()` method, which
returns the time in UNIX timestamp format.

```php
$timestamp = $bucket->getLastModified('file.txt');
```

### Mime Type

To get the file mime type, use the `getMimeType()` method, which returns a string mime type representation of the file.

```php
$mime = $bucket->getMimeType('file.txt');
```

### Visibility

In addition to such characteristics as file existence, there is also file visibility. You can get
information about the visibility of a file using the `getVisibility()` method, and update the visibility using the
`setVisibility()` method.

These methods operate on constants that have been defined in the `Spiral\Storage\Visibility` enum-like interface.
So, the sample code with file visibility control will look like the following.

```php
$visibility = $bucket->getVisibility('file.txt');

// If file is "private" then publish it
if ($visibility === Visibility::VISIBILITY_PRIVATE) {
    $bucket->setVisibility('file.txt', Visibility::VISIBILITY_PUBLIC);
}
```

> **Note**
> In case of using a bucket located on Windows OS, this functionality may not work.

### URI Publishing

The storage component initially provides only operations for working with files on arbitrary file systems. However,
some systems, in addition to storage, allow HTTP to organize an access point to such files to receive their contents
through a browser.

The storage component allows you to add an arbitrary URI resolver of public addresses for these files using
the [distribution component](../component/distribution.md).

To configure a resolver, you should familiarize yourself with the configuration of this component. After that, for a
specific bucket, simply add a section containing a link to a specific distribution resolver.

Each bucket is able to specify the distribution.

```php
return [
    // ...
    'buckets' => [
        'uploads' => [
            'server' => '...',

            //
            // + Add relation to existing distribution.
            //
            'distribution' => 'NAME_OF_DISTRIBUTION'
        ],
    ],
];
```

To use this URI resolver, simply call the `toUri()` method. In case you need any other distribution, you can specify it
explicitly in the `toUriFrom(...)`method.

> **Note**
> Unlike other methods, this one can only be called on a specific file.

```php
use Spiral\Storage\BucketInterface;
use Spiral\Distribution\UriResolverInterface;

class UriController
{
    // Using default URI resolver
    public function getUri(BucketInterface $bucket): string
    {
        return (string)$bucket
            ->file('picture.jpg')
            ->toUri();
    }

    // Using another URI resolver
    public function getAnotherUri(BucketInterface $bucket, UriResolverInterface $resolver): string
    {
        return (string)$bucket
            ->file('picture.jpg')
            ->toUriFrom($resolver);
    }
}
```

Some generators may accept additional options. Such arguments can be passed to the `toUri([...$arguments])`
or `toUriFrom($resolver, [...$arguments])` methods. For example, if you create a link to the CloudFront, you can
additionally specify the expiration time of this link.

You can read more about possible additional arguments on the corresponding page
of the [distribution documentation](../component/distribution.md).

```php
$uri = $file->toUri(new \DateInterval('PT30S'));
// And
$uri = $file->toUriFrom($resolver, new \DateInterval('PT30S'));
```

# Distribution

The `spiral/distribution` component is responsible for providing public HTTP links on arbitrary resources. In most
cases, this will be the same address as the address of the site itself, however, in some cases, resources may be located
on external servers such as [Amazon CloudFront](https://aws.amazon.com/cloudfront/) or some other CDN. In these cases,
generating a public link to the resource needs to use a specific API of the provider, or write one's own code for the
used
CDN. The component makes this interaction easier and provides a number of built-in drivers for generating URIs to
external suppliers.

## Installation

Use Composer to install the component:

```terminal
composer require spiral/distribution
```

### Framework Integration

> **Note**
> Please note that the spiral/framework >= 2.8 already includes this component.

To enable the component, you just need to add the `Spiral\Distribution\Bootloader\DistributionBootloader` class to the
bootloader list, which is located in the class of your application.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    // Added distribution bootloader
    \Spiral\Distribution\Bootloader\DistributionBootloader::class,
    // ...
];
```

## Configuration

The configuration file for this component looks like the one below. Just create a `distribution.php` file and add it to
the
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
[the container](../framework/container.md) or
via [dependency injection](../framework/container.md#dependency-injection).

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

Unlike a similar method used to generate an address for a page in the [url generator](../http/routing.md#url-generation)
router component, links can be arbitrary and configured on a separate server designed to serve static content.

In this way, if you pass an arbitrary file string to the `resolve()` method, you will receive a physical http link to
this
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
required in the parameters of the constructor, you should use [the bootloader](../framework/bootloaders.md).
