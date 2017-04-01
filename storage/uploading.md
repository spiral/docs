# Uploading Files
As mention in previous sections storage manager support `put` command by resources, filename or `StreamInterface` or `UploadedFileInterface`.

You can put uploaded files directly to remote location using following code:

```php
public function uploadAction(StorageInterface $storage, InputManager $input)
{
    $object = $storage->put('uploads', 'file.name', $input->file('upload'));
    dump($object->getAddress());
}
```

Please note that it is recommended to put your uploads to local bucket prior to remote one if you want to work to pre-process your file (for example generate an image):

```php
public function uploadAction(StorageInterface $storage, InputManager $input)
{
    $object = $storage->put('local', 'file.name', $input->file('upload'));

    $this->doSomething($object);

    $object->replace('cloud');

    dump($object->getAddress());
}
```