# Testing — Storage Tests

Spiral provides is a convenient way to test the [spiral/storage](../advanced/storage.md) component.

The `fakeStorage` method allows you to create a fake storage that mimics the behavior of a real storage, but doesn't
actually send any files to the cloud. This way, you can test file uploads without worrying about accidentally sending
real files to the cloud.

```php
use Tests\TestCase;

final class UserServiceTest extends TestCase
{
    private \Spiral\Testing\Storage\FakeBucket $bucket;

    protected function setUp(): void
    {
        parent::setUp();
        $this->bucket = $this->fakeStorage()->bucket('avatars');
    }


    public function testUserShouldBeRegistered(): void
    {
        // Perform user registration ...

        $this->bucket->assertCreated('avatars/john_smith.jpg');
    }
}
```

You can then use the assertion methods provided by the class, such as `assertExists`, `assertCreated`,
`assertNotExist`, `assertNotCreated`, etc to check if specific file exists in the bucket.

### Asserting File Existence

The `assertExists` method asserts that the specified file exists in the bucket.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertExists('image.jpg');
```

### Asserting File non-Existence

The `assertNotExist` method asserts that the specified file does not exist in the bucket.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotExist('image.jpg');
```

### Asserting File Creation

The `assertCreated` method asserts that the specified file was created in the bucket.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertCreated('image.jpg');
```

### Asserting File non-Creation

The `assertNotCreated` method asserts that the specified file was not created in the bucket.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotCreated('image.jpg');
```

### Asserting File Deletion

The `assertDeleted` method asserts that the specified file was deleted from the bucket.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertDeleted('image.jpg');
```

### Asserting File non-Deletion

The `assertNotDeleted` method asserts that the specified file was not deleted from the bucket.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotDeleted('image.jpg');
```

### Asserting File Move

The `assertMoved` method asserts that the specified file was moved from one location to another.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertMoved('file.txt', 'folder/file.txt');
```

### Asserting File non-Move

The `assertNotMoved` method asserts that the specified file was not moved from one location to another.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotMoved('file.txt', 'folder/file.txt');
```

### Asserting File Copy

The `assertCopied` method asserts that the specified file was copied from one location to another.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertCopied('file.txt', 'folder/file.txt');
```

### Asserting File non-Copy

The `assertNotCopied` method asserts that the specified file was not copied from one location to another.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotCopied('file.txt', 'folder/file.txt');
```

### Asserting File Visibility

The `assertVisibilityChanged` method asserts that the specified file has the given visibility.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertVisibilityChanged('file.txt');
```

### Asserting File Visibility not changed

The `assertVisibilityNotChanged` method asserts that the specified file has not changed its visibility.

```php
$uploads = $storage->bucket('uploads');
$uploads->assertVisibilityNotChanged('file.txt');
```

