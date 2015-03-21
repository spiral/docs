# Components and traits
Spiral Component is base implementation for most of spiral core classes, controllers, components and models.
Implementation of component is fairly simple and includes only 2 methods. And one constant.
```php
class Component
{
    const SINGLETON = '';
    public static function getAlias();
    public static function make($parameters = array());
}
```
Method `getAlias` used to correctly resolve name of logger/event dispatcher binded to parent component. Default
implementation of that method will return class name, however method can be redefined to return custom value.

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
Test::make(); //Will return instance of Test2
```
Additionally `make` method inject provided parameters in class constructor.
```php
class Test extends Component 
{
    public function __constuct($name, Core $core)
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
> Attention, extending class with change `getAlias` method value, which will require system to dedicate new
`Logger` instance. To keep same logger instance for component and it's all child - redefine getAlias method.

Spiral logger is PSR-3 based, means you can replace it with your custom implementation or external library, 
such as `Monolog`.

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
    $event->content = strtoupper($event->context);
    $event->stopPropagation(); //Ignore other events
});

$class = MyClass::make(); //or new MyClass();
echo $class->doAction('value'); //dump of class + "VALUE"
```
> Check Core\Events section for more information about events.

### SingletonTrair


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