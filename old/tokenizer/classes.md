# Class Location
Spiral tokenizer provide way to locate classes based on parent, interface or used trait.

Request `Spiral\Tokenizer\ClassesInterface` as dependency to work with such methods.

## Examples

Locate all controllers in application:

```php
public function indexAction(ClassesInterface $classes)
{
    dump($classes->getClasses(ControllerInterface::class));
}
```

Locate all RequestFilters:

```php
public function indexAction(ClassesInterface $classes)
{
    dump($classes->getClasses(RequestFilter::class));
}
```

Locate all classes with support of container shortcuts:

```php
public function indexAction(ClassesInterface $classes)
{
    dump($classes->getClasses(SharedTrait::class));
}
```