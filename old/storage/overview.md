# Storage Manager
The Spiral `StorageManager` (`Spiral\Storage\StorageInterface`) component provides a simple and unified abstraction layer (SAL) across multiple data/file storages including local filesystem (`FilesInterface` based), Amazon S3, Rackspace Files, remote server over SFTP, remote server over FTP, GridFS storage.

The general concept of the StorageManager is to abstract stored data/files using only two primary entities: objects and **buckets** and limit set of operations as much as it can.

> The third storage entity **server** is hidden inside buckets and not considered something the developer should have access to.

Storage objects (`StorageObject` or `ObjectInterface` class) are only responsible for high level abstraction including metadata wrapping and exposing set of helper methods. Buckets (`StorageBucker` or `BucketInterface`) however handle all low level data operations, such as adding, removing, renaming and replacing objects in bucket or buckets.

> StorageManager is tightly integrated with PSR7 streams and the `UploadedFileInterface`.

You can read more about working with [objects and buckets here](/old/storagerage/entities.md) and read how to configure storage servers in [this section](/old/storagerage/servers.md).

## Storage Identification
Unlike other storage implementations Spiral SAL layer is based upon one universal object address created by combining bucket prefix and filename. Such combination is treated as object address and, in some cases, might be send directly to user.

## Configuring StorageBuckets
StorageManager configuration and association between buckets, prefixes and storages are located in storage config:

```php
return [
    /*
     * Storage server configurations.
     */
    'servers' => [
        'local'     => [
            'class'   => Servers\LocalServer::class,
            'options' => [
                'home' => directory('runtime')
            ]
        ],
        'amazon'    => [
            'class'   => Servers\AmazonServer::class,
            'options' => [
                'accessKey' => '',
                'secretKey' => '',
            ]
        ],
        'rackspace' => [
            'class'   => Servers\RackspaceServer::class,
            'options' => [
                'username' => '',
                'apiKey'   => ''
            ]
        ],
        'ftp'       => [
            'class'   => Servers\FtpServer::class,
            'options' => [
                'host'     => '127.0.0.1',
                'login'    => 'username',
                'password' => '',
                'home'     => '/'
            ]
        ],
        'sftp'      => [
            'class'   => Servers\SftpServer::class,
            'options' => [
                'host'       => 'hostname.com',
                'home'       => '/home/',
                'authMethod' => 'pubkey',
                'username'   => '',
                'password'   => '',
                'publicKey'  => '',
                'privateKey' => ''
            ]
        ],
        'gridFS'    => [
            'class'   => Servers\GridFSServer::class,
            'options' => [
                'database' => 'default'
            ]
        ],
    ],

    /*
     * Buckets define target locations for files/data to be stored in. Each bucket must have associated
     * prefix and server.
     */
    'buckets' => [
        'local'     => [
            'server'  => 'local',
            'prefix'  => 'local:',
            'options' => [
                //Directory has to be specified relatively to root directory of associated server
                'directory' => 'storage/'
            ]
        ],
        'amazon'    => [
            'server'  => 'amazon',
            'prefix'  => 'https://s3.amazonaws.com/bucketName/',
            'options' => [
                'public' => true,
                'bucket' => 'bucketName'
            ]
        ],
        'rackspace' => [
            'server'  => 'rackspace',
            'prefix'  => 'rackspace:',
            'options' => [
                'container' => 'container-name',
                'region'    => 'DFW'
            ]
        ],
        'ftp'       => [
            'server'  => 'ftp',
            'prefix'  => 'ftp:',
            'options' => [
                'directory' => '/',
                'mode'      => \Spiral\Files\FilesInterface::RUNTIME
            ]
        ],
        'sftp'      => [
            'server'  => 'sftp',
            'prefix'  => 'sftp:',
            'options' => [
                'directory' => 'uploads',
                'mode'      => \Spiral\Files\FilesInterface::RUNTIME
            ]
        ],
        'gridFS'    => [
            'server'  => 'gridFS',
            'prefix'  => 'gridFS:',
            'options' => [
                'collection' => 'files'
            ]
        ],
    ]
];
```

### Configuration Walk-thought
Configuration file declares 6 servers we can use to store data in, every server has it's unique name, adapter class and set of connection options. 

Follow [this section](/old/storagerage/servers.md) to find out how to properly configure each individual server.

## Multiple Environment
The biggest benefits of using StorageManager is that you can point your buckets to different servers in different application environments. For example your staging might store user uploads at local hard-drive, when production instances send files to the cloud.
 
> Tip: do not forget to handle `StorageException` when performing storage related operations.