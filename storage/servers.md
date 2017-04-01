# Storage servers
The StorageManager component support multiple servers and buckets for each of the server. Server connection configuration is located in `storage` config and can be altered in runtime by talking to `StorageInterface`.

## Local Server
`FilesInterface` stores data on local hard-drive. It has only one global option "home" which every bucket directory will be related to, bucket options must include storage directory and optional mode (by default RUNTIME - 755).

Example server definition:

```php
'local'     => [
    'class'   => Servers\LocalServer::class,
    'options' => [
        'home' => directory('runtime')
    ]
],
```

Example bucket definition:

```php
'local'     => [
    'server'  => 'local',
    'prefix'  => 'local:',
    'options' => [
        'directory' => 'storage/' // runtime/storage/
    ]
],
```

## Amazon S3 
`AmazonServer` talks to AWS S3 storage via Guzzle package.

Example server definition:

```php
'amazon'    => [
    'class'   => Servers\AmazonServer::class,
    'options' => [
        'accessKey' => '',
        'secretKey' => '',
    ]
],
```

> You can include addition arguments such as `timeout` (default 0) and `server` (default "https://s3.amazonaws.com");

In your buckets you have to specify AWS S3 bucket name and public/private flag.

```php
'amazon'    => [
    'server'  => 'amazon',
    'prefix'  => 'https://s3.amazonaws.com/my-bucket/',
    'options' => [
        'public' => true,
        'bucket' => 'my-bucket'
    ]
],
```

Setting "public" option to true will make every uploaded file available for access from web without signed requests.

## Rackspace Files
`RackspaceServer` works similar to AWS, though, due protocol difference, it have different credential management system.

Example server definition:

```php
'rackspace' => [
    'class'   => Servers\RackspaceServer::class,
    'options' => [
        'username' => '',
        'apiKey'   => ''
    ]
],
```

Additional options available:

```php
$options = [
    'server'     => 'https://auth.api.rackspacecloud.com/v1.0',
    'authServer' => 'https://identity.api.rackspacecloud.com/v2.0/tokens',
    'username'   => '',
    'apiKey'     => '',
    'cache'      => true,   //Enable to use cache to store credentials and tokens
    'lifetime'   => 86400   //Cache lifetime
];
```

Rackspace buckets requires us to define region bucket located in (Rackspace has limited abilities of copying files from buckets located in different regions), example:

```php
'rackspace' => [
    'server'  => 'rackspace',
    'prefix'  => 'rackspace:',
    'options' => [
        'container' => 'container-name',
        'region'    => 'DFW'
    ]
],
```

> You can find bucket region in Rackspace control panel.

## FTP server
`FtpServer` stores it's files on remote server:

```php
'ftp'       => [
    'class'   => Servers\FtpServer::class,
    'options' => [
        'host'     => '127.0.0.1',
        'login'    => 'Wolfy-J',
        'password' => '',
        'home'     => '/'
    ]
],
```

Additional server options:

```php
$options = [
    'host'     => '',
    'port'     => 21,
    'timeout'  => 60,
    'login'    => '',
    'password' => '',
    'home'     => '/',
    'passive'  => true
];
```

Bucket definition is similar to `LocalServer`:

```php
'sftp'      => [
    'server'  => 'sftp',
    'prefix'  => 'sftp:',
    'options' => [
        'directory' => 'uploads',
        'mode'      => \Spiral\Files\FilesInterface::RUNTIME
    ]
],
```

## SFTP server
`SftpServer` works almost identically to `FtpServer` but over SSH:

```php
'sftp'      => [
    'class'   => Servers\SftpServer::class,
    'options' => [
        'host'       => 'hostname.com',
        'home'       => '/home/',
        'authMethod' => 'pubkey', //
        'username'   => '',
        'password'   => '',
        'publicKey'  => '',
        'privateKey' => ''
    ]
],
```

Additional options and SFTP authorization:

```php
/**
 * Authorization methods.
 */
const NONE     = 'none';
const PASSWORD = 'password';
const PUB_KEY  = 'pubkey';

/**
 * @var array
 */
protected $options = [
    'host'       => '',
    'methods'    => [],
    'port'       => 22,
    'home'       => '/',
    
    //Authorization method and username
    'authMethod' => 'password',
    'username'   => '',
    
    //Used with "password" authorization
    'password'   => '',
    
    //User with "pubkey" authorization
    'publicKey'  => '',
    'privateKey' => '',
    'secret'     => null
];
```

Bucket definition:

```php
'sftp'      => [
    'server'  => 'sftp',
    'prefix'  => 'sftp:',
    'options' => [
        'directory' => 'uploads',
        'mode'      => \Spiral\Files\FilesInterface::RUNTIME
    ]
],
```

## GridFS server
You can also use MongoDB to store your files:

```php
'gridFS'    => [
    'class'   => Servers\GridFSServer::class,
    'options' => [
        //Works with default database if any, use new Autowire (db bingin) to use custom db
    ]
],
```

Example bucket definition:

```php
'gridfs'    => [
    'server'  => 'gridfs',
    'prefix'  => 'gridfs:',
    'options' => [
        'collection' => 'files'
    ]
]
```

Use ODM extension to support 

## More Sevres
Implement your own storage server or server bridge by implementing `ServerInterface`:

```php
interface ServerInterface
{
    /**
     * Check if object exists at server under specified bucket. Must return false if object does not
     * exists.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     *
     * @return bool
     * @throws ServerException
     */
    public function exists(BucketInterface $bucket, string $name): bool;

    /**
     * Get object size in specified bucket or return false.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     *
     * @return int|null
     * @throws ServerException
     */
    public function size(BucketInterface $bucket, string $name);

    /**
     * Put object data into specified bucket under given name, must replace existed data.
     *
     * @param BucketInterface                 $bucket
     * @param string                          $name
     * @param string|StreamInterface|resource $source
     *
     * @return bool
     * @throws ServerException
     */
    public function put(BucketInterface $bucket, string $name, $source): bool;

    /**
     * Must return filename which is valid in associated FilesInterface instance. Must trow an
     * exception if object does not exists. Filename can be temporary and should not be used
     * between sessions.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     *
     * @return string
     * @throws ServerException
     */
    public function allocateFilename(BucketInterface $bucket, string $name): string;

    /**
     * Return PSR7 stream associated with bucket object content or trow and exception.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     *
     * @return StreamInterface
     * @throws ServerException
     */
    public function allocateStream(BucketInterface $bucket, string $name): StreamInterface;

    /**
     * Delete bucket object if it exists.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     *
     * @throws ServerException
     */
    public function delete(BucketInterface $bucket, string $name);

    /**
     * Rename storage object without changing it's bucket.
     *
     * @param BucketInterface $bucket
     * @param string          $oldName
     * @param string          $newName
     *
     * @return bool
     * @throws ServerException
     */
    public function rename(BucketInterface $bucket, string $oldName, string $newName): bool;

    /**
     * Copy storage object to another bucket. Both buckets must belong to same server.
     *
     * @param BucketInterface $bucket
     * @param BucketInterface $destination
     * @param string          $name
     *
     * @return bool
     * @throws ServerException
     */
    public function copy(BucketInterface $bucket, BucketInterface $destination, string $name): bool;

    /**
     * Move storage object data to another bucket. Both buckets must belong to same server.
     *
     * @param BucketInterface $bucket
     * @param BucketInterface $destination
     * @param string          $name
     *
     * @return bool
     * @throws ServerException
     */
    public function replace(
        BucketInterface $bucket,
        BucketInterface $destination,
        string $name
    ): bool;
}
```

Spiral StorageManager has been written originally in 2010-2011, you can find a lot of alternative libraries since then, for example [Flysystem](https://github.com/thephpleague/flysystem).