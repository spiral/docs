# Storage Manager
Spiral `StorageManager` (`Spiral\Storage\StorageInterface`) component provides unified abstraction across multiple data/file storages including local harddrive (`FilesInterface` used), Amazon S3, Rackspace Files, remove server over SFTP, remote server over FTP, GridFS storage. General principle of StorageManager is to abstract stored data/files using only two primary enties: objects and **buckets**. The third type or entity - **server** are hidden inside buckets.

In this two entities, objects (`StorageObject` or `ObjectInterface` class) are only responsible for high level abstraction including metadata wrapping and exposing set of helper methods. Bucket (`StorageBucker` or `BucketInterface`) hovewer reposible for low level data operations, such as adding, removing, renaming and replacing objects in bucket or buckets.

> StorageManager has deep intergration with PSR7 streams and `UploadedFileInterface`.

## StorageInterface
First of all, let's view our primary storage interface understand what set of methods are available for us:

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

## Storage Buckets
StorageManager buckets provides set of operations you might want to consider use in your application:

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

One the most important properties of StrorageBuckets is **prefix** and **server**. Prefix responsible for generating storage object address unique for it's storage bucket, when server are responsible for all low level bucket operations.

> As you can see there is no directory related set of operation as you can encode directory name into object name.

## Configuring StorageBuckets


## Using StorageManager
As with other components you can get access to storage manager using `Spiral\Storage\StorageInterface` or short binding "storage". 


