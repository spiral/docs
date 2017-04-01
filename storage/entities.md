# Storage Manager
Once your storages is configure we can start working with our files.

## Interfaces
Let's look as `BucketInterface` and `ObjectInterface` to see what functionality is available for us:

```php
interface BucketInterface
{
    /**
     * Bucket name.
     *
     * @return string
     */
    public function getName(): string;

    /**
     * Associated storage server instance.
     *
     * @return ServerInterface
     *
     * @throws StorageException
     */
    public function getServer(): ServerInterface;


    /**
     * Get bucket version with some options changed.
     *
     * $bucket->withOption('public', false)->put(...)
     *
     * @param string $name
     * @param mixed  $value
     *
     * @return self
     */
    public function withOption(string $name, $value): BucketInterface;

    /**
     * Get server specific bucket option or return default value.
     *
     * @param string $name
     *
     * @param null   $default
     *
     * @return mixed
     */
    public function getOption(string $name, $default = null);

    /**
     * Get bucket prefix.
     *
     * @return string
     */
    public function getPrefix(): string;

    /**
     * Check if address be found in bucket namespace defined by bucket prefix.
     *
     * @param string $address
     *
     * @return bool|int Should return matched address length (change in future).
     */
    public function hasAddress(string $address);

    /**
     * Build object address using object name and bucket prefix. While using URL like prefixes
     * address can appear valid URI which can be used directly at frontend.
     *
     * @param string $name
     *
     * @return string
     */
    public function buildAddress(string $name): string;

    /**
     * Check if given name points to valid and existed location in bucket server.
     *
     * @param string $name
     *
     * @return bool
     *
     * @throws BucketException
     */
    public function exists(string $name): bool;

    /**
     * Get object size or return false if object not found.
     *
     * @param string $name
     *
     * @return int|null
     *
     * @throws BucketException
     */
    public function size(string $name);

    /**
     * Put given content under given name in associated bucket server. Must replace already existed
     * object.
     *
     * @param string                                              $name
     * @param string|StreamInterface|StreamableInterface|resource $source String can only be
     *                                                                    filename.
     *
     * @return string Return inserted object address.
     *
     * @throws BucketException
     */
    public function put(string $name, $source): string;

    /**
     * Must return filename which is valid in associated FilesInterface instance. Must trow an
     * exception if object does not exists. Filename can be temporary and should not be used
     * between sessions.
     *
     * @param string $name
     *
     * @return string
     *
     * @throws BucketException
     */
    public function allocateFilename(string $name): string;

    /**
     * Return PSR7 stream associated with bucket object content or trow and exception.
     *
     * @param string $name Storage object name.
     *
     * @return StreamInterface
     *
     * @throws BucketException
     */
    public function allocateStream(string $name): StreamInterface;

    /**
     * Delete bucket object if it exists.
     *
     * @param string $name Storage object name.
     *
     * @throws BucketException
     */
    public function delete(string $name);

    /**
     * Rename storage object without changing it's bucket. Must return new address on success.
     *
     * @param string $oldName
     * @param string $newName
     *
     * @return string
     *
     * @throws StorageException
     * @throws BucketException
     */
    public function rename(string $oldName, string $newName): string;

    /**
     * Copy storage object to another bucket. Method must return address which points to
     * new storage object.
     *
     * @param BucketInterface $destination
     * @param string          $name
     *
     * @return string
     *
     * @throws BucketException
     */
    public function copy(BucketInterface $destination, string $name): string;

    /**
     * Move storage object data to another bucket. Method must return new object address on success.
     *
     * @todo Add ability to specify new name, not only destination.
     *
     * @param BucketInterface $destination
     * @param string          $name
     *
     * @return string
     *
     * @throws BucketException
     */
    public function replace(BucketInterface $destination, string $name): string;
}
```

```php
interface ObjectInterface extends StreamableInterface
{
    /**
     * Get object name inside parent bucket.
     *
     * @return string
     */
    public function getName(): string;

    /**
     * Get full object address.
     *
     * @return string
     */
    public function getAddress(): string;

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
     * @throws ObjectException
     */
    public function exists(): bool;

    /**
     * Get object size or return false of object does not exists.
     *
     * @return int|bool
     * @throws ObjectException
     */
    public function getSize();

    /**
     * Must return filename which is valid in associated FilesInterface instance. Must trow an
     * exception if object does not exists. Filename can be temporary and should not be used
     * between sessions. You must never write anything to this file.
     *
     * @return string
     * @throws ObjectException
     */
    public function localFilename(): string;

    /**
     * Delete object from associated bucket.
     *
     * @throws ObjectException
     */
    public function delete();

    /**
     * Rename storage object without changing it's bucket.
     *
     * @param string $newName
     *
     * @return self
     * @throws ObjectException
     */
    public function rename(string $newName): ObjectInterface;

    /**
     * Copy storage object to another bucket. Method must return ObjectInterface which points to
     * new storage object.
     *
     * @param BucketInterface|string $destination
     *
     * @return self
     * @throws ObjectException
     */
    public function copy($destination): ObjectInterface;

    /**
     * Move storage object data to another bucket.
     *
     * @param BucketInterface|string $destination
     *
     * @return self
     * @throws ObjectException
     */
    public function replace($destination): ObjectInterface;

    /**
     * Must be serialized into object address.
     *
     * @return string
     */
    public function __toString(): string;
}
```

## Put New file into bucket
We can put new file/data into specified bucket using either the `StorageManager` or the `BucketInterface` put method, we are going to use `StorageManager` received using short binding "storage" in controller action:

```php
protected function indexAction()
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

> You are not able to store strings in buckets due security reasons, use streams and resources instead.

As result of such code we will get file named "file.txt" in our "app/runtime/storage" directory. In object dump you may notice that address is specified as "local:file.txt", such address is created using object name and bucket prefix and provides us ability to clearly identify file location.

## Open storage object by it's address
Once we have data located in some bucket we only need to store generated address somewere as it's value clearly defines object name and object bucket. Let's use address from previous example and try to get some information about our object:

```php
protected function indexAction()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);
    dump($object);
}
```
We can read stored file information without worrying where file is stored:

```php
protected function indexAction()
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

## Access file content
To read stored file content use functions `getSteam` and `localFilename`:

```php
protected function indexAction()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);

    //Get PSR7 stream associated with object data
    dump($object->getStream());
    
    //Get local filename associated with object
    dump($object->localFilename());
}
```

If `getStream` method is self explanatory and can be used, for example, in generating PSR7 response, `localFilename` must be described.

First of all, `localFilename` method will return READ-ONLY filename. 
You can this filename in copy, read and other file operations like in provided example:

```php
protected function indexAction()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);
    
    //Get local filename associated with object
    $this->files->copy($object->localFilename(), 'local.txt');
}
```

You must know that given filename is not necessary real filename at all, in many cases (when you working with remote servers like Amazon or Rackspace) it will look like `spiral://000000000f6cae89000000005103baf7` and only be accessible inside spiral environment via stream wrapper. 

## Rename object
We can easily rename of our storage object using method `rename`:

```php
protected function indexAction()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);

    $object->rename('new.txt');
}
```

All storage servers support ability to automatically create directory for storage objects, meaning we can include our folder into object name:

```php
protected function indexAction()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);

    $object->rename('folder/new.txt');
}
```

## Delete object
Any storage object can be deleted at any moment using `delete` method:

```php
protected function indexAction()
{
    $address = 'local:file.txt';
    $object = $this->storage->open($address);
    $object->delete();
}
```

## Move or Copy object to another bucket
One of them most powerful features of storage manager is ability to copy or move storage object into specified bucket without worrying about how it will be done. Object address will be corrected based on it's parent bucket: 

```php
protected function indexAction()
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

> Spiral will try to move and copy files in a most efficient way as possible, for example copying file from one amazon bucket to another will happen on amazon size. However in many cases (especially in cross server copying) current application memory will be used as copy/move buffer.
