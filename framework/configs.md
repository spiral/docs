# Injectable Configs
Spiral framework components do not call configuration files directly, instead they are requesting dependency injection for component specific configuration decorator. Default framework bundle provides ability to automatically route and populate such decorators using contextual injections.

## Create configuration
In some cases you might want to create configuration for your component, service or module. First step will be to locate php file with needed confugration in app/config, let's name this file "my-config.php":

```php
return [
    'option' => 'value'
];
```

You can now get access to your configuration by requesting ConfiguratorInterface in your contructor or method injections:

```php
public function indexAction(ConfiguratorInterface $configurator)
{
    dump($configurator->getConfig('my-config')['option']);
}
```

This method might work in some cases, however spiral ships with more convinient way to work with config files.

## Injectable Configs
In many cases presenting your configuration files can give you more benefits (for example improve testability):

```php
class MyConfig extends InjectableConfig
{
    /**
     * This constant will tell Configurator which config section we are going to decorate.
     */
    const CONFIG = 'my-config';

    /**
     * We only need this property to remember needed structure.
     *
     * @var array
     */
    protected $config = [
        'option' => null
    ];

    /**
     * @return string
     */
    public function getOption()
    {
        return $this->config['option'];
    }
}
```

Now you can simply request this class as dependency in your code:

```php
public function indexAction(MyConfig $config, ConfiguratorInterface $configurator)
{
    dump($config->getOption());
    dump($config->getOption() === $configurator->getConfig('my-config')['option']);
}
```

> Representing config as class can give other benefits, for example we can update our components and handle legacy config values in this class, not inside our component.

```php
public function getOption()
{
    if (!empty($this->config['new-option'])) {
        //Some new feature
        return $this->config['new-option'];
    }

    return $this->config['option'];
}
```

> There is many configs across spiral which you can use in your code, for example HttpConfig. Since Configurator will cache already loaded decorators in memory there no much performance drop on requesting config dependency in multiple places.
