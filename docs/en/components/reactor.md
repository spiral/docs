# Code Generation
The Spiral Reactor component includes set of OOP based code declarations used to create classes, methods using declarative approach.

This section will review basic Reactor usages.

> See [spiral/scaffolder](https://github.com/spiral-modules/scaffolder) to code generation in action.

## Generate controller
Let's generate controller declaration with pre-defined action method:

```php
$controller = new ClassDeclaration('TestController', Controller::class);
$controller->property('service')
    ->setAccess(ClassDeclaration\PropertyDeclaration::ACCESS_PRIVATE)
    ->setDefault(null)
    ->setComment('@var SomeService');

$method = $controller->method('indexAction');
$method->setAccess(ClassDeclaration\MethodDeclaration::ACCESS_PUBLIC);

$method->parameter('value')->setType('string')->setDefault(null);

$method->setSource([
    'return $this->service->doSomething($value);'
]);

dump($controller->render());
```

Resulted code:

```php
class TestController extends Spiral\Core\Controller
{
    /**
     * @var SomeService
     */
    private $service = null;

    public function indexAction(string $value = null)
    {
        return $this->service->doSomething($value);
    }
}
```