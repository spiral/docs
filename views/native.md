# Views - Plain PHP
You can use plain php files as your views without any compilation or cache layer. This engine is enabled by default
and available in Web application skeleton.

## Create view
To create plain PHP view simply put file with `.php` extension into `views` directory. Template can be rendered by it's 
filename:

```php
<?php // test.php
echo "hello world";
```

```php
public function index(ViewsInterface $views)
{
    return $views->render('test'); // no need to specify the extension
}
```

All the passed parameters will be available as PHP variables:

```php
public function index(ViewsInterface $views)
{
    return $views->render('test', ['name' => 'Antony']); 
}
```

Where `test.php`:

```php
Hello , <?= $name ?>!
```

## Accessing Container
You can access the container in your view files via `$this->container`:

```php
Hello world, <?= $name ?>!

<?php dump($this->container->get(MyService::class)); ?>
```
