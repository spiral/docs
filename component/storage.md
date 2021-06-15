# Storage

Component `spiral/storage` provides a powerful storage abstraction thanks
to the wonderful [Flysystem](https://github.com/thephpleague/flysystem) PHP
package by Frank de Jonge. The Storage component integration provides simple
drivers for working with local filesystems and Amazon S3. Even better, it's
amazingly simple to switch between these storage options between your local
development machine and production server as the API remains the same for each
system.

Please note that, unlike classical file systems, the store component provides
an API which provides the operations of writing a file, checking its existence,
reading and getting the public address of this file. All operations
(which classic file systems have) for working with directories, or a list of
files are not available.

To install the component:

```bash
$ composer require spiral/storage
```

## Framework Integration

> Please note that the spiral/framework >= 2.8 already includes this component.

Make sure to add `Spiral\Bootloader\Storage\StorageBootloader` to your App class:

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Storage\StorageBootloader::class
];
```

## Configuration

Storage config file should be located (by default) at `app/config/storage.php`.
Within this file, you may configure the servers (the "`servers`" section) and
the specific storages (the "`buckets`" section) that your servers will use and
which you will use.

For example, the simplest configuration with one server and two storages might
look like this:

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

    'default' => 'uploads',

    /**
     * -------------------------------------------------------------------------
     *  Storage Servers
     * -------------------------------------------------------------------------
     *
     * Here are each of the servers is configured for your application. Of
     * course, examples of customizing each available server supported by
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
     * Here are a list of specific buckets (or storages) that use the above
     * server settings. Each "server" section in this list must refer to a
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

> Please note that this configuration is only available when used with a
> Spiral Framework.

### Manual Configuration (Outside The Framework)

This way of using the component is required only if it is installed separately,
outside the framework.

First you need to create a storage instance where all your buckets will
be stored. After that, you can add and get arbitrary bucket from it by the
desired name.

```php
$storage = new \Spiral\Storage\Storage();

$storage->add('example', \Spiral\Storage\Bucket::fromAdapter(
    new \League\Flysystem\Local\LocalFilesystemAdapter(__DIR__ . '/path/to/directory')
));

$file = $storage->bucket('example')
    ->write('file.txt', 'content');
```

As you may have noticed, you can use existing
[flysystem adapters](https://flysystem.thephpleague.com/v2/docs/adapter/local/)
to create a bucket. Just install the one you want and add it to the store using
the `Bucket::fromAdapter()` method.

### Local Server

The local server, as the name implies, is located in the local file system:
In the same place where the executable code of your application is located.

You have already seen an example of local server settings earlier, however, to
simplify them, some optional sections have been specially removed. Let's now
take a look at the complete configuration of this type of server, leaving all
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

This type of server is designed to interact with an external distributed file
system using the S3 protocol. S3 is the main protocol for communicating with
[Amazon](https://s3.console.aws.amazon.com/s3/home) servers. In addition to
Amazon S3 itself, there are free alternatives that you can install and use on
your own server, for example [Minio Server](https://docs.min.io/).

To set up this type of server, you will need an existing bucket and personal
authentication data. In addition, for completeness, the example below will
include optional parameters that are optional and have default values.

Please note that in order to interact with this type of servers, you must have
one of the two available packages installed (either). You must install the
`league/flysystem-aws-s3-v3` or `league/flysystem-async-aws-s3` package using
the Composer.

```bash
$ composer require league/flysystem-aws-s3-v3 ^2.0
// OR
$ composer require league/flysystem-async-aws-s3 ^2.0
```

During configuration, you should specify which of the packages you will use in
the server's "adapter" section. The `"s3"` value corresponds to the
`league/flysystem-aws-s3-v3` package, while the `"s3-async"` value corresponds 
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
            // Optional key of S3 API endpoint URI. This value is required when
            // using a server other than Amazon.
            //
            'endpoint' => env('S3_ENDPOINT', null),

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
            // (Available since v2.9).
            // Any additional options you want to pass to S3 client that are not
            // listed above (e.g. 'use_path_style_endpoint').
            //
            'options' => [],
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
            // In the case that you want to use another bucket using the main
            // server settings, then you can redefine it by specifying the
            // appropriate configuration key.
            //
            'bucket' => env('S3_BUCKET', null),

            //
            // If the new bucket is in a different region, then you can also
            // override this value.
            //
            'region' => env('S3_REGION', null),

            //
            // A similar way can be done with the directory prefix in cases
            // where a particular bucket must refer to some other root directory.
            //
            'prefix' => 'custom-directory',
        ]
    ]
];
```

### Custom Server

In some cases, standard adapters may not be enough and in this case you may
need to specify your own. You can also use your config file to configure your
custom adapter.

In this case, the adapter section must refer to the
`League\Flysystem\FilesystemAdapter` implementation, and the "options" section
will contain an array of arguments passed to the constructor of this adapter.

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

Finally, after we have familiarized ourselves with what types of servers the
component supports, we can move on to using them.

The storage architecture assumes 3 different levels of access to identical
operations: Storage, Bucket and File. At each of these levels, you can operate
on files, but the differences are in how much data you have to transfer to a
particular method. At the highest "Storage" level you have to pass information
about the bucket and the file, at the "Bucket" level only about the file, and
finally at the "File" level you will not have to pass any additional information.

In practice, it will look like this. Let's try to create a file `example.txt` in
3 different ways inside some controller of our application.

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

- When working with a storage, you should pass the file name in the URI-like
  format `[BUCKET_NAME]://[FILE_NAME]`. In all methods of working with storage,
  this is the first string argument.

- In all methods of working with a bucket, this is a file name in an arbitrary
  format. Please also note that the leading slash does not affect the file
  location in any way and the names `file.txt` and `/file.txt` will be
  completely identical.
  
- In all methods of working with a specific file, no additional arguments are
  required.


Each of the methods has both its advantages and disadvantages. Just use what
you like best.

Since in the example above we are using a controller, then at the same time we
will use an alternative dependency, which is suitable in cases of simple file
saving, when many implementations of buckets are not required.

In such cases, as below, the bucket used in the system by default will be
selected. You can define this bucket in the `'default'` section of your
configuration.

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

After we have familiarized ourselves with the options for using the same methods
at different levels of the store, we should go on to describe the possibilities
directly. Let's start!

### Create And Write

In case to create a file, you can use one of the two available methods:
`create()` or `write()`. The first creates an empty file in cases where it is
not created, and the second allows you to additionally write arbitrary `string`
or `resource` stream content there.

For example, the code that creates the file might look like this.

```php
// Creating file from bucket
$file = $bucket->create('file.txt');

// Creating file from bucket with string content
$file = $bucket->write('file.txt', 'message');

// Creating file from bucket with resource stream content
$file = $bucket->write('file.txt', fopen(__DIR__ . '/local/file.txt', 'rb+'));
```

### Copy And Move

To copy files, use the `copy()` method, which contains one required argument with
the name of the new file and one optional - the bucket where this file should
be copied. If the second argument is not specified, then the compilation bucket
will be identical to the original one.

The `move()` method is completely similar to using the `copy()` method, but 
instead of copying it moves the file.

```php
$backup = $firstBucket->copy('from.txt', 'backup.txt');

$moved  = $firstBucket->move('backup.txt', 'to.txt', $secondBucket);
```

> Please note that when a file is moved (or copied) from a bucket with private
> permissions to a distribution with public permissions, the visibility of the
> file will also change to the one corresponding to this bucket.

### Delete

To delete a file, just use the `delete()` method. This method accepts an
optional boolean argument meaning deletion an empty directory in which this
file was located.

```php
$bucket->delete('file.txt');
```

### Reading Content

There are two ways to read the contents of an existing file: Using the
`getContents()` and `getStream()` methods. The first one returns the string
content of the file, and the second resource is a stream for working with
streaming data.

```php
$string = $bucket->getContents('text.txt');

$resource = $bucket->getStream('music.mp3');
```

### Existence Check

To check the existence of a file, use the `exists()` method, which returns a
boolean value `true` if the file exists in the bucket, or `false` if it does
not exist.

```php
$isExists = $bucket->exists('file.txt');
```

### File Size

To get information about the size of a file, use the `getSize()` method, which
returns the size of an existing file in bytes.

```php
$bytes = $bucket->getSize('file.txt');
```

### Last Modification Time

To retrieve information about the date of the last modification of a file, use
the `getLastModified()` method, which returns the time in UNIX timestamp format.

```php
$timestamp = $bucket->getLastModified('file.txt');
```

### Mime Type

To get the file mime type, use the `getMimeType()` method, which returns a 
string mime type representation of the file.

```php
$mime = $bucket->getMimeType('file.txt');
```

### Visibility

In addition to such characteristics as the existence of a file, there is also
the visibility of the file. You can get information about the visibility of a
file using the `getVisibility()` method, and update the visibility using the
`setVisibility()` method.

These methods operate on constants that have been defined in the
`Spiral\Storage\Visibility` enum-like interface. Thus, the sample code with
file visibility control will look like the following.

```php
$visibility = $bucket->getVisibility('file.txt');

// If file is "private" then publish it
if ($visibility === Visibility::VISIBILITY_PRIVATE) {
    $bucket->setVisibility('file.txt', Visibility::VISIBILITY_PUBLIC);
}
```

> Please note that in the case of using a bucket located on Windows OS, this
> functionality may not work.

### URI Publishing

The storage component initially provides only operations for working with files
on arbitrary file systems. However, some systems, in addition to storage, allow
HTTP to organize an access point to such files to receive their contents through
a browser.

The storage component allows you to add an arbitrary URI resolver of public
addresses for these files using the
[distribution component](/component/distribution.md).

To configure the resolver, you should familiarize yourself with the
configuration of this component. After that, for a specific bucket, simply add
a section containing a link to a specific distribution resolver.

Each bucket has such an opportunity to specify the distribution.

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

To use this URI resolver, simply call the `toUri()` method. In case you need any
other distribution, you can specify it explicitly in the
`toUriFrom(...)`method.

> Please note that, unlike other methods, this one can only be called on a 
> specific file.

```php
use Spiral\Storage\BucketInterface;
use Spiral\Distribution\UriResolverInterface;

class UriController
{
    // Using default URI resovler
    public function getUri(BucketInterface $bucket): string
    {
        return (string)$bucket
            ->file('picture.jpg')
            ->toUri();
    }

    // Usign another URI resolver
    public function getAnotherUri(BucketInterface $bucket, UriResolverInterface $dist): string
    {
        return (string)$bucket
            ->file('picture.jpg')
            ->toUriFrom($dist);
    }
}
```
