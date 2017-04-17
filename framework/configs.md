# Injectable Configs
All framework components talk to config files though the set of config objects. Framework bundle provides ability to automatically route and populate such objects using contextual injections by `ConfigFactory`.

## Create configuration
To create new config object, place file in app/config (sub-dirs are allowed), example "my-config.php":

```php
return [
    'option' => 'value'
];
```

Config data can now be read using ConfiguratorInterface:

```php
public function indexAction(ConfiguratorInterface $configurator)
{
    dump($configurator->getConfig('my-config')['option']);
}
```

## ConfigObject
In order to properly represent configuration in the container create class extends `InjectableConfig`:

```php
class MyConfig extends InjectableConfig
{
    const CONFIG = 'my-config';

    //Default config values (docs) if any
    protected $config = [
        'option' => null
    ];

    /**
     * @return string
     */
    public function getOption(): string
    {
        return $this->config['option'];
    }
}
```

Config object can be used immediately as injection:

```php
public function indexAction(MyConfig $config, ConfiguratorInterface $configurator)
{
    dump($config->getOption());
    dump($config->getOption() === $configurator->getConfig('my-config')['option']);
}
```

Config objects can be used to properly handle legacy configuration formats:

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

> All created configs are cached in `ConfigFactory`, use `flushCache()` to reset such cache.
