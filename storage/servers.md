# Storage servers
Spiral StorageManager support multiple servers to be used to store you data. Every server must be configured with connection/basic options and must have correlated bucket declare few adapter specific options.

## Local Server
Local server utilizes local hard drive and `FilesInterface` to store your data. It has only one global option used to declare "home" storage directory - every bucket located will be realted to such home, bucket options must include storage directory and optional mode (by default RUNTIME - 777).

Example server definition:

```php
'local'     => [
    'class'   => Servers\LocalServer::class,
    'options' => [
        'home' => directory('root')
    ]
],
```

Example bucket definition:

```php
'local'     => [
    'server'  => 'local',
    'prefix'  => 'local:',
    'options' => [
        'directory' => '/application/runtime/storage/'
    ]
],
```

## Amazon S3 
Amazon S3 storage server works with amazon api using Guzzle clients and requires to specify connection credentials on server level and amazon container name and access policy on bucket level.

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

You can also specify two additinal options for amazon server - timeout (0 by default) and server ("https://s3.amazonaws.com" by default).

```php
protected $options = [
    'server'    => 'https://s3.amazonaws.com',
    'timeout'   => 0,
    'accessKey' => '',
    'secretKey' => ''
];
```

Example bucket definition:

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

> Amazon and Rackspace servers was written originally in 2011, if you wish to update their code to use official SDK - you will be much appriciated.

## Rackspace Files
Rackspace server uses same Guzzle package to perform requests, however due rackspace requires additional query to fetch available buckets and generate access token it declares additional dependency with CacheStore to be remember such token and buckets list.

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

Let's view additional options we can use:

```php
protected $options = [
    'server'     => 'https://auth.api.rackspacecloud.com/v1.0',
    'authServer' => 'https://identity.api.rackspacecloud.com/v2.0/tokens',
    'username'   => '',
    'apiKey'     => '',
    'cache'      => true,   //Enable to use cache to store credentials and tokens
    'lifetime'   => 86400   //Cache lifetime
];
```

Rackspace buckets requires us to define region bucket located in (Rackspace has limited abilities of copying files from buckets located in different regions), example 

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
FTP storage server provides ability to utilize FTP protocol to store files on remote machine. As in case with local server it provides ability to specify home directory for every server bucket.

Example server definition:

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

Let's view additional server options:

```php
protected $options = [
    'host'     => '',
    'port'     => 21,
    'timeout'  => 60,
    'login'    => '',
    'password' => '',
    'home'     => '/',
    'passive'  => true
];
```

As in case with files we are able to specify default file mode for bucket:

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
Sftp server are very similar to FTP and utilized ssh2 php extension to connect to remove machines. Server support multiple authorization ways, default one - public key, as in case with FTP or local server we can specify home directory on server level:

```php
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
```

Let's review additional server options and authorization methods:

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

Example bucket definition:

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
GridFS server connects to mongo using spiral ODM, you should only declare what database should be used on server level and what collection on bucket level.

Example server definition:

```php
'gridfs'    => [
    'class'   => Servers\GridfsServer::class,
    'options' => [
        'database' => 'default'
    ]
]
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

## Add more servers
If you wish to create custom storage server you must only implement `Spiral\Storage\ServerInterface`, you can access to bucket specific options using `$bucket->option()` method.

```php
interface ServerInterface
{
    /**
     * Check if object exists at server under specified bucket. Must return false if object does not
     * exists.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     * @return bool
     * @throws ServerException
     */
    public function exists(BucketInterface $bucket, $name);

    /**
     * Get object size in specified bucket or return false.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     * @return int|bool
     * @throws ServerException
     */
    public function size(BucketInterface $bucket, $name);

    /**
     * Put object data into specified bucket under given name, must replace existed data.
     *
     * @param BucketInterface        $bucket
     * @param string                 $name
     * @param string|StreamInterface $source
     * @return bool
     * @throws ServerException
     */
    public function put(BucketInterface $bucket, $name, $source);

    /**
     * Must return filename which is valid in associated FilesInterface instance. Must trow an
     * exception if object does not exists. Filename can be temporary and should not be used
     * between sessions.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     * @return string|bool
     * @throws ServerException
     */
    public function allocateFilename(BucketInterface $bucket, $name);

    /**
     * Return PSR7 stream associated with bucket object content or trow and exception.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     * @return StreamInterface
     * @throws ServerException
     */
    public function allocateStream(BucketInterface $bucket, $name);

    /**
     * Delete bucket object if it exists.
     *
     * @param BucketInterface $bucket
     * @param string          $name
     * @throws ServerException
     */
    public function delete(BucketInterface $bucket, $name);

    /**
     * Rename storage object without changing it's bucket.
     *
     * @param BucketInterface $bucket
     * @param string          $oldname
     * @param string          $newname
     * @return bool
     * @throws ServerException
     */
    public function rename(BucketInterface $bucket, $oldname, $newname);

    /**
     * Copy storage object to another bucket. Both buckets must belong to same server.
     *
     * @param BucketInterface $bucket
     * @param BucketInterface $destination
     * @param string          $name
     * @return bool
     * @throws ServerException
     */
    public function copy(BucketInterface $bucket, BucketInterface $destination, $name);

    /**
     * Move storage object data to another bucket. Both buckets must belong to same server.
     *
     * @param BucketInterface $bucket
     * @param BucketInterface $destination
     * @param string          $name
     * @return bool
     * @throws ServerException
     */
    public function replace(BucketInterface $bucket, BucketInterface $destination, $name);
}
```

Spiral StorageManager has been written originally in 2010-2011, since then there is a lot of nice libraries and new ways to store your files remotelly. One of the notable implementation is [Flysystem] (https://github.com/thephpleague/flysystem). Flysystem can be easily intergated into StorageManager due it provides much lower abstraction level than required by StorageManager, such implementation can significantly improve list of supported storages (for example if you want Dropbox support).
