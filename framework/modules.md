# Making Modules
Spiral framework will be useless without an ability to alter it's core functionality, connect extensions, add resources or visual plugins (widgets/tags). Any of this features can be achieved using ModuleManager and Bootloaders.

## Writing Bootloader
In some cases your module might need to define set of container bindings or bootload specific code, in this case you can simply use technique described in [bootloaders](/framework/bootloaders.md) section.

> In some cases (for example, when you package contains only widgets or delcarative singletons) you might skip Bootloader creation for your module.

## Writing Module Class
Usually your module code goes with set of resources to be added to application or set of configs to be mounted. Both of this operations can be performed using module class which can be created by simply implementing `Spiral\Modules\ModuleInterface`:

```php
class MyModule implements ModuleInterface
{
    /**
     * {@inheritdoc}
     */
    public function register(RegistratorInterface $registrator)
    {

    }
    
    /**
     * {@inheritdoc}
     */
    public function publish(PublisherInterface $publisher, DirectoriesInterface $directories)
    {

    }
}
```

### Publish method
Public method in your module are responsible for moving and publishing files into application, for example you can move configurations or some assets using directories interface aliases:

```php
/**
 * {@inheritdoc}
 */
public function publish(PublisherInterface $publisher, DirectoriesInterface $directories)
{
  $publisher->publish(__DIR__ . 'configs/config.php', $directories->directory('config'));

  //You can either publish whole directory content
  $publisher->publishDirectory(
    __DIR__ . '../resources/',
    $directories->directory('webroot') . 'resources/'
  );
}
```

## Installing, Registering and Publishing your module
i'm writing it


> Check spiral [toolkit](https://github.com/spiral/toolkit) or [profiler](https://github.com/spiral/profiler) repositories to get some examples.
