# Native Views
You can use plain php files as your views without any compilation or cache layer.

## Accessing Container inside view
You can access `Spiral\Views\Engines\Native\NativeView` instance inside your template.

```php
Hello world, <?= $name ?>!

<?php dump($this->container->get(MyService::class)); ?>
```

NativeView declares IoC scope for static container, you can use `app` or `spiral`:

```php
Hello world, <?= $name ?>!

<?= spiral(MyService::class)->someMethod(); ?>
<?= spiral('faker')->someMethod(); ?>
<?= app()->faker->name ?>
```