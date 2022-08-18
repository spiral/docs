# Views - Plain PHP

You can use plain php files as your views without any compilation or cache layer. This engine enabled by default and
available in the Web application skeleton.

## Create view

To create plain PHP view simply put file with `.php` extension into `views` directory. Template can be rendered by it's
filename:

```php
<?php // test.php
echo "hello world";
```

```php
public function index(ViewsInterface $views): string
{
    return $views->render('test'); // no need to specify the extension
}
```

All the passed parameters will be available as PHP variables:

```php
public function index(ViewsInterface $views): string
{
    return $views->render('test', ['name' => 'Antony']); 
}
```

Where `test.php`:

```php
Hello , <?= $name ?>!
```

## Container

You can access the container in your view files via `$this->container`:

```php
Hello world, <?= $name ?>!

<?php dump($this->container->get(MyService::class)); ?>
```

## Global variables

You can add global variables that will be available in all views. There are several ways to add a global variable.
Using configuration file `app/config/views.php`.

```php
return [
    'globalVariables' => [
        'some_var' => env('SOME_VALUE'),
        'other_var' => 'other_value'
    ]
];
```

Or you can use `Spiral\Views\GlobalVariablesInterface`.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\GlobalVariablesInterface ;
use Spiral\Boot\EnvironmentInterface;

class AppBootloader extends Bootloader 
{
    public function boot(GlobalVariablesInterface $vars, EnvironmentInterface $env): void
    {
         $vars->set('some_var', $env->get('SOME_VALUE'));
         $vars->set('other_var', 'other_value');
    }
}
```

After that, you can use the variable in the view.

```php
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <?=$some_var?>

    <?=$other_var?>
</body>
</html>
```
