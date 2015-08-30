# Storage Manager
Spiral `StorageManager` (`Spiral\Storage\StorageInterface`) component provides unified abstraction across multiple data/file storages including local harddrive (`FilesInterface` used), Amazon S3, Rackspace Files, remove server over SFTP, remote server over FTP, GridFS storage.

General concept of StorageManager is to abstract stored data/files using only two primary enties: objects and **buckets** and limit set of operations as much as it can. 

> The third storage entity **server** are hidden inside buckets and not concidered as something developer should have access to.

In this two entities, objects (`StorageObject` or `ObjectInterface` class) are only responsible for high level abstraction including metadata wrapping and exposing set of helper methods. Buckets (`StorageBucker` or `BucketInterface`) hovewer handle all low level data operations, such as adding, removing, renaming and replacing objects in bucket or buckets.

> StorageManager has deep intergration with PSR7 streams and `UploadedFileInterface`.

## StorageInterface
First of all, let's view our primary storage interface to understand what set of methods are available for us:

```php
interface StorageInterface
{
    /**
     * Register new bucket using it's options, server id and prefix.
     *
     * @param string $name
     * @param string $prefix
     * @param string $server
     * @param array  $options
     * @return BucketInterface
     * @throws StorageException
     */
    public function registerBucket($name, $prefix, $server, array $options = []);

    /**
     * Get bucket by it's name.
     *
     * @param string $bucket
     * @return BucketInterface
     * @throws StorageException
     */
    public function bucket($bucket);

    /**
     * Find bucket instance using object address.
     *
     * @param string $address
     * @param string $name Name stripped from address.
     * @return BucketInterface
     * @throws StorageException
     */
    public function locateBucket($address, &$name = null);

    /**
     * Get or create instance of storage server.
     *
     * @param string $server
     * @param array  $options Used to create new instance.
     * @return ServerInterface
     * @throws StorageException
     */
    public function server($server, array $options = []);

    /**
     * Put object data into specified bucket under provided name. Should support filenames, PSR7
     * streams and streamable objects. Must create empty object if source empty.
     *
     * @param string|BucketInterface                    $bucket
     * @param string                                    $name
     * @param mixed|StreamInterface|StreamableInterface $source
     * @return ObjectInterface|bool
     * @throws StorageException
     * @throws BucketException
     * @throws ServerException
     */
    public function put($bucket, $name, $source = '');

    /**
     * Create instance of storage object using it's address.
     *
     * @param string $address
     * @return ObjectInterface
     * @throws StorageException
     * @throws ObjectException
     */
    public function open($address);
}
```

As you can see StorageManager provides us ability to get specific bucket, server or object instance or create new one of each. Let's try to review buckets functionality first.

## Storage Buckets
StorageManager buckets provides set of operations you might want to consider using in your application, to start let's check `BucketInterface`:

```php
interface BucketInterface
{
    /**
     * @param string           $server  Responsible server id or name.
     * @param string           $prefix  Bucket prefix.
     * @param array            $options Server related options.
     * @param StorageInterface $storage
     * @param FilesInterface   $files
     */
    public function __construct(
        $server,
        $prefix,
        array $options,
        StorageInterface $storage,
        FilesInterface $files
    );

    /**
     * Get server specific bucket option or return default value.
     *
     * @param string $name
     * @param null   $default
     * @return mixed
     */
    public function getOption($name, $default = null);

    /**
     * Get server name or ID associated with bucket.
     *
     * @return string
     */
    public function getServerID();

    /**
     * Associated storage server instance.
     *
     * @return ServerInterface
     * @throws StorageException
     */
    public function server();

    /**
     * Get bucket prefix.
     *
     * @return string
     */
    public function getPrefix();

    /**
     * Check if address be found in bucket namespace defined by bucket prefix.
     *
     * @param string $address
     * @return bool|int Should return matched address length.
     */
    public function hasAddress($address);

    /**
     * Build object address using object name and bucket prefix. While using URL like prefixes
     * address can appear valid URI which can be used directly at frontend.
     *
     * @param string $name
     * @return string
     */
    public function buildAddress($name);

    /**
     * Check if given name points to valid and existed location in bucket server.
     *
     * @param string $name
     * @return bool
     * @throws ServerException
     * @throws BucketException
     */
    public function exists($name);

    /**
     * Get object size or return false if object not found.
     *
     * @param string $name
     * @return int|bool
     * @throws ServerException
     * @throws BucketException
     */
    public function size($name);

    /**
     * Put given content under given name in associated bucket server. Must replace already existed
     * object.
     *
     * @param string                                     $name
     * @param string|StreamInterface|StreamableInterface $source
     * @return ObjectInterface
     * @throws ServerException
     * @throws BucketException
     */
    public function put($name, $source);

    /**
     * Must return filename which is valid in associated FilesInterface instance. Must trow an
     * exception if object does not exists. Filename can be temporary and should not be used
     * between sessions.
     *
     * @param string $name
     * @return string
     * @throws ServerException
     * @throws BucketException
     */
    public function allocateFilename($name);

    /**
     * Return PSR7 stream associated with bucket object content or trow and exception.
     *
     * @param string $name Storage object name.
     * @return StreamInterface
     * @throws ServerException
     * @throws BucketException
     */
    public function allocateStream($name);

    /**
     * Delete bucket object if it exists.
     *
     * @param string $name Storage object name.
     * @throws ServerException
     * @throws BucketException
     */
    public function delete($name);

    /**
     * Rename storage object without changing it's bucket. Must return new address on success.
     *
     * @param string $oldname
     * @param string $newname
     * @return string|bool
     * @throws StorageException
     * @throws ServerException
     * @throws BucketException
     */
    public function rename($oldname, $newname);

    /**
     * Copy storage object to another bucket. Method must return address which points to
     * new storage object.
     *
     * @param BucketInterface $destination
     * @param string          $name
     * @return string
     * @throws ServerException
     * @throws BucketException
     */
    public function copy(BucketInterface $destination, $name);

    /**
     * Move storage object data to another bucket. Method must return new object address on success.
     *
     * @param BucketInterface $destination
     * @param string          $name
     * @return string
     * @throws ServerException
     * @throws BucketException
     */
    public function replace(BucketInterface $destination, $name);
}
```

One the most important properties of StrorageBuckets is **prefix** and **server**. Prefix responsible for generating storage object address unique for it's parent bucket by joining object name and bucket prefix values; server are responsible for all low level bucket operations. You can also notice dependency of `FilesInterface`, this dedendency are required due buckets provides us bridge between local file enviroment and remote storegae.

> As you can see there is no directory related operation as you can encode directory name into object name.

One very important bucket function called located in `getOption` method, this method much provide set of server specific options required to perform low level operations, usually such options declared in storage component configuration file (`application/config/storage.md`).

> You can read more about server specific options in [storage servers] (servers.md) section.

## Configuring StorageBuckets
Before we will jump to some of our examples, let's try to declare and configure few buckets and related servers, we can use storage configuration file to archive that:

```php
return [
    'servers' => [
        'local'     => [
            'class'   => Servers\LocalServer::class,
            'options' => [
                'home' => directory('root')
            ]
        ],
        'amazon'    => [
            'class'   => Servers\AmazonServer::class,
            'options' => [
                'verify'    => false,
                'accessKey' => '',
                'secretKey' => '',
            ]
        ],
        'rackspace' => [
            'class'   => Servers\RackspaceServer::class,
            'options' => [
                'verify'   => false,
                'username' => '',
                'apiKey'   => ''
            ]
        ],
        'ftp'       => [
            'class'   => Servers\FtpServer::class,
            'options' => [
                'host'     => '127.0.0.1',
                'login'    => 'Wolfy-J',
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
        'gridfs'    => [
            'class'   => Servers\GridfsServer::class,
            'options' => [
                'database' => 'default'
            ]
        ]
    ],
    'buckets' => [
        'local'     => [
            'server'  => 'local',
            'prefix'  => 'local:',
            'options' => [
                //Temporary location
                'directory' => '/application/runtime/storage/'
            ]
        ],
        'uploads'   => [
            'server'  => 'local',
            'prefix'  => '/uploads/',
            'options' => [
                //Temporary location
                'directory' => '/webroot/uploads/'
            ]
        ],
        'amazon'    => [
            'server'  => 'amazon',
            'prefix'  => 'https://s3.amazonaws.com/my-bucket/',
            'options' => [
                'public' => true,
                'bucket' => 'my-bucket'
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
        'gridfs'    => [
            'server'  => 'gridfs',
            'prefix'  => 'gridfs:',
            'options' => [
                'collection' => 'files'
            ]
        ]
    ]
];
```

First of all, out configuration file declares 6 servers we can use to store data in, every server has it's unique name, adapter class and set of connection options, in our case we declared: local, amazon, rackspace, ftp, sftp and gridfs servers. You can read more about server configurations in dedicated configuration section.

Next, we created few named containers each associated with unique prefix, server and server specific options, in our case we have: 
* **local** - will store data in local directory "application/runtime/storage" (directory will be created automatically). Work with local filesystem directly. Evert created object will gain prefix "local:" (such prefix can not be understand by frontend and expose real file location).
* **uploads** - stores data in public directory "webroot/uploades" with prefix "/uploads/", such prefix provides us ability to send object address directory to frontend or view.
* **amazon** - utilized Amazon S3 server storage with prefix "https://s3.amazonaws.com/my-bucket/" (such prefix provides us ability to send object address to frontend directly). As additional server options we stated that every bucket file must be public.
* **rackspace** - uses Rackspace Files server with default region "DFW" and specified rackspace bucket name. Our prefix - "rackspace:".
* **ftp** - provides us access to remote server over ftp connection, points directly to server root directory and specified default file mode as 777. Prefix - "ftp:" (can not be exposed to client).
* **sftp** - remove server storage over sftp connection, points to "/home/uploads" directory with private prefix "sftp:" and default file mode 777.
* **gridfs** - stores data in MongoDB GridFS collection "files", private prefix "gridfs:".

## Work with Storage component
Once we have our buckets and servers configured (you must enter your own connection values in servers you wish to use) we can start working with our storage component. We are going to use provided config as example.

### Put New file into bucket
We can put new file/data into specified bucket using `StorageManager` or `BucketInterface` put method, we are going to use `StorageManager` received using short binding "storage" in controller action:

```php
public function index()
{
    //We can put data into bucket using filename
    $object = $this->storage->put('local', 'file.txt', __FILE__);
    dump($object);

    //Resource
    $object = $this->storage->put('local', 'file.txt', fopen(__FILE__, 'rb'));
    dump($object);

    //Stream
    $object = $this->storage->put('local', 'file.txt', \GuzzleHttp\Psr7\stream_for(
        fopen(__FILE__, 'rb')
    ));
    dump($object);

    //Or even using uploaded file
    if (!empty($file = $this->input->file('upload'))) {
        $object = $this->storage->put('local', 'file.txt', $file);
        dump($object);
    }
}
```

> Every put operation will replace existed data.

As result of such code we will get file named "file.txt" in our "application/runtime/storage" directory. In object dump you may notice that address is specified as "local:file.txt", such address is created using object name and bucket prefix and provides us ability to crealy identify file location.

### Open storage object by it's address
Once we have data located in some bucket we only need to store generated address somewere as it's value clearly defines object name and object bucket. Let's use address from previous example and try to get some information about our object:

```php
public function index()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);
    dump($object);
}
```

Before we will start working with retrieved object, let's quickly look as `ObjectInterface` to check what methods are available for us:

```php
interface ObjectInterface extends StreamableInterface
{
    /**
     * @param string           $address Full object address.
     * @param StorageInterface $storage Storage component.
     * @throws ObjectException
     */
    public function __construct($address, StorageInterface $storage);

    /**
     * Get object name inside parent bucket.
     *
     * @return string
     */
    public function getName();

    /**
     * Get full object address.
     *
     * @return string
     */
    public function getAddress();

    /**
     * Get associated bucket instance.
     *
     * @return BucketInterface
     */
    public function getBucket();

    /**
     * Check if object exists.
     *
     * @return bool
     * @throws ServerException
     * @throws BucketException
     * @throws ObjectException
     */
    public function exists();

    /**
     * Get object size or return false of object does not exists.
     *
     * @return int|bool
     * @throws ServerException
     * @throws BucketException
     * @throws ObjectException
     */
    public function getSize();

    /**
     * Must return filename which is valid in associated FilesInterface instance. Must trow an
     * exception if object does not exists. Filename can be temporary and should not be used
     * between sessions. You must never write anything to this file.
     *
     * @return string
     * @throws ServerException
     * @throws BucketException
     * @throws ObjectException
     */
    public function localFilename();

    /**
     * Delete object from associated bucket.
     *
     * @throws ServerException
     * @throws BucketException
     * @throws ObjectException
     */
    public function delete();

    /**
     * Rename storage object without changing it's bucket.
     *
     * @param string $newname
     * @return self
     * @throws ServerException
     * @throws BucketException
     * @throws ObjectException
     */
    public function rename($newname);

    /**
     * Copy storage object to another bucket. Method must return ObjectInterface which points to
     * new storage object.
     *
     * @param BucketInterface|string $destination
     * @return self
     * @throws ServerException
     * @throws BucketException
     * @throws ObjectException
     */
    public function copy($destination);

    /**
     * Move storage object data to another bucket.
     *
     * @param BucketInterface|string $destination
     * @return self
     * @throws ServerException
     * @throws BucketException
     * @throws ObjectException
     */
    public function replace($destination);

    /**
     * Must be serialized into object address.
     *
     * @return string
     */
    public function __toString();
}
```

Now we can modify our code to read object properties without being worrying where object is stored into:

```php
public function index()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);

    dump($object->exists());
    dump($object->getName());
    dump($object->getSize());
    dump($object->getAddress());
    dump($object->getBucket());
    
    dump($object->getStream());
    dump($object->localFilename());
}
```

From given methods let's focus on two most important: `getStream` and `localFilename`.

### Accessing storage object data
Storage component will be useless without providing you ability to read data of stored object. Right now you have 2 methods use can use your our application:

```php
public function index()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);

    //Get PSR7 stream associated with object data
    dump($object->getStream());
    
    //Get local filename associated with object
    dump($object->localFilename());
}
```

If `getStream` method is self explanatory and can be used, for example, in generating PSR7 response, `localFilename` must be described closesly.

First of all, `localFilename` method will return filename which must never be used to write any data into, it is purely read only. You can easily use such
filename in copy, read and other file operations like in provided example:

```php
public function index()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);
    
    //Get local filename associated with object
    $this->files->copy($object->localFilename(), 'local.txt');
}
```

Hovewer you must know that given filename is not nesessary real filename at all, in many cases (when you working with remote servers like Amazon or Rackspace) it will look like `spiral://000000000f6cae89000000005103baf7` and only be accessible inside spiral enviroment. Such behaviour presented due spiral will create "virtual" filenames for some storages using stream wrapper instead of writing object data to real local filename.

> Recevied local filename will be automatically descructed, do not remove it by youself.

### Rename object
We can easily rename of our storage object using method `rename`:

```php
public function index()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);

    $object->rename('new.txt');
}
```

All storage servers support ability to automatically create directory for storage objects, meaning we can include our folder into object name:

```php
public function index()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);

    $object->rename('folder/new.txt');
}
```

### Delete object
Any storage object can be deleted at any moment using `delete` method:

```php
public function index()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);
    $object->delete();
}
```

### Move or Copy object to another bucket
One of them most poweverful features of storage manager is ability to copy or move storage object into specified bucket without worrying about how it will be done. Object address will be corrected based on it's parent bucket: 

```php
public function index()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);
    
    //Object will return self from "replace method" and new object from "copy" method
    
    //Replacing from "local" to "uploads": uploads/file.txt
    dump($object->replace('uploads')->getAddress());
    
    //Replacing from "uploads" to "amazon": https://s3.amazonaws.com/my-bucket/file.txt
    dump($object->replace('amazon')->getAddress());
    
    //Replacing from "amazon" to "rackspace": rackspace:file.txt
    dump($object->replace('rackspace')->getAddress());

    //Replacing from "rackspace" to "ftp": ftp:file.txt
    dump($object->replace('ftp')->getAddress());
    
    //Replacing from "ftp" to "sftp": sftp:file.txt
    dump($object->replace('sftp')->getAddress());
    
    //Replacing from "sftp" to "gridfs": gridfs:file.txt
    dump($object->replace('gridfs')->getAddress());
    
    //Returning to original location
    dump($object->replace('local')->getAddress());
    
    //Copying file to amazon
    dump($object->copy('amazon')->getAddress());
}
```

> Spiral will try to move and copy files in a most efficient way as possible, for example copying file from one amazon bucket to another will happen on amazon size. Hovewer in many cases (especially in cross server copying) current application memory will be used as copy/move buffer.

## Multiple application enviroments
One of the biggest benefits of using StorageManager is that you can point your buckets to different servers in different application enviroments. Due your StorageManager code are pretty universal you can have your bucket pointing to local hard drivr in development and, for example, to Amazon S3 in production. In addtion to that you can easity change your application storage logic at any moment by simply introducing more buckets.

> Tip: do not forget to handle `StorageException` when performing storage related operations.

> Tip: use local bucket to keep files for some processing before sending to permanent storage, for example every uploaded image can be stored on server directly, processed in background and send to permanent bucket after, in this case you can avoid any network charges caused by downloading file from remote storage to process it.
