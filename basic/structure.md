# Application Structure
Framework does not enforce any specific namespace or directory structure, you can freely change them.

# Directories
The directory structure can be controlled via `mapDirectories` method of Kernel class. By
default, all application directories will be calculated based on `root` using the following pattern:

Directory | Value 
---       | ---
root      | **set by user**
vendor   | **root**/vendor
runtime  | **root**/runtime
cache    | **root**/runtime/cache
public   | **root**/public
app      | **root**/app
config   | **root**/config
resources| **app**/resources

You can specify `root` or any other directory value in your `app.php` file as argument to `App`:

```php
$app = \App\App::init(['root' => __DIR__]);
```

For example we can change runtime directory location:

```php
$app = \App\App::init(['root' => __DIR__, 'runtime' => sys_get_temp_dir()]);
```

To resolve directory alias within your application use `Spiral\Boot\DirectoriesInterface`:

```php
function test(DirectoriesInterface $dirs)
{
    dump($dirs->get('app'));
}
```

You can also use function `directory` inside global IoC scope (config files, controllers, service code).

```php
function test()
{
    dump(directory('app'));
}
```

## Namespaces
By default, all skeleton applications use `App` root namespace pointing to `app/src` directory. You can change base 
to any desired namespace in `composer.json`:

```json
{
  "autoload": {
    "psr-4": {
      "App\\": "app/src/"
    }
  }
}
```
