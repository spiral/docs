# Components and traits
Spiral Component is base implementation for most of spiral core classes, controllers, components and models.
Implementation of component is fairly simple and includes only 2 methods and one contant.
```php
class Component
{
    const SINGLETON = '';
    public static function getAlias();
    public static function make($parameters = array());
}
```
Method `getAlias` used to correctly resolve name of logger/event dispatcher binded to parent component. Default
implementation of that method will return class name, hovewer method can be redefined to return custom value.

Method `make` will automatically bypass call to spiral `Container` to create instance of requested component
with regard to declared bindings.
```php
class Test extends \Spiral\Core\Component
{
}

class Test2 extends Test
{
}

Core::bing("Test", "Test2");
Test::getAlias();  //Test
Test2::getAlias(); //Test2
Test::make();      //Return instance of Test2
```
Additionally `make` method inject provided parameters in class constructor.
```php
class Test extends Component 
{
    public function __construct($name, Core $core)
    {
    }
}

Test::make(["name" => "value"]); //Core will be injected automatically by Container
```
> Using this way to create instances of components is not required and makes sense only when class implementation
can be potentially altered by developer on project basis.

## Component Traits
Spiral provides set of traits specially designed to work with `Component` classes and relaying on `getAlias` value.

### LoggerTrait
LoggerTrait provides on demand access to `Logger` instance binded with component using it's `alias` value.
Logger associated statically and can be accessed even without constructing class.
```php
use Spiral\Core\Component\LoggerTrait;

class MyClass extends Component
{
    use LoggerTrait;
}

//Instance of Spiral\Components\Debug\Logger
MyClass::logger()->info("Some message, some {value}", ["value" => "abc"]);

//We can replace Logger for all component instances
MyClass::setLogger(new Monolog\Logger("name"));

$class = new MyClass();
$class->logger()->warning("Some message, some {value}", ["value" => "abc"]); //Instance of Monolog\Logger
```
> Attention, extending class will change `getAlias` method value, which will require system to dedicate new
`Logger` instance. To keep same logger instance for component and it's all child - redefine getAlias method.

### EventsTrait
Used to provide component based event dispatcher. This trait used in `DBAL\Database`, `DataEntity`, `Validator`
and other classes.
```php
use Spiral\Core\Component\EventsTrait;
use Spiral\Core\Events\ObjectEvent;

class MyClass extends Component
{
    use EventsTrait;
    
    public function doAction($context)
    {   
        //Perform `doAction` event with given context, ObjectEvent will be used 
        //to represent event walker
        return  $this->event('doAction', $context);
    }
}

MyClass::dispatcher()->addListener('doAction', function (ObjectEvent $event) 
{
    dump($event->object); //Instance of MyClass
    $event->context = strtoupper($event->context);
    $event->stopPropagation(); //Ignore other events
});

$class = MyClass::make(); //or new MyClass();
echo $class->doAction('value'); //dump of class + "VALUE"
```
> Check Core\Events section for more information about events.

### SingletonTrait
Singleton trait used in most of spiral core components such as `DBAL`, `ODM`, `Files` and etc. Trait is not
pure `Singleton` pattern as constructed instance stored in Container scope and can be removed or replaced at
any moment. You have to define constant `SINGLETON` to make it work.
```php
use Spiral\Core\Component\SingletonTrait;

class Test extends Component
{
    use SingletonTrait;
    
    const SINGLETON = __CLASS__;
}
```
Now singleton can be received using 4 different methods:
```php
public function action(Test $testA)
{
    $testB = Test::make();
    $testC = Test::getInstance();
    $testD = Container::get('Test');
    
    assert($testA === $testB);
    assert($testB === $testC);
    assert($testC === $testD);
    assert($testA instanceof Test);
}
```
You can always desctuct signleton by using `Container::removeBinding()` method.
```php
Container::removeBinding('Test'); //No more Test instance
```
> You can use custom `SINGLETON` constant value, however in this case additional core binding are required.


Singletons follow container bindings:
```php
Core::bind("Test", "Test2");
```
```php
public function action(Test $testA, Test2 $testE)
{
   $testB = Test::make();
    $testC = Test::getInstance();
    $testD = Container::get('Test');
    $testF = Container::get('Test2');

    assert($testA === $testB);
    assert($testB === $testC);
    assert($testC === $testD);
    assert($testD === $testE);
    assert($testE === $testF);

    assert($testA instanceof Test2);
}
```
But behaviour of `getAlias` method will be changed:
```php
Test::getAlias();  //Test
Test2::getAlias(); //Test (!)
```
## Convert existed class to Component
You can convert existed class to become component by simply adding trait `Spiral\Core\Component\ComponentTrait`.
```php
use Spiral\Core\Component\ComponentTrait;
use Spiral\Core\Component\LoggerTrait;

class MyClass extends SomeParent 
{
    use ComponentTrait, LoggerTrait;
}
```