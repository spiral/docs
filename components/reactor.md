# Reactor
Reactor component does not considered to be part of spiral components and exists only in spiral framework bundle. This component provides ability to generate classes with desired structure and functionaly of logic level avoiding using templates, such ability provides flexible way to create scaffolding functionality for different framework parts.

Reactor can be separated by two general pieces: elements and generators.

## Reactor Elements
Reactor element provides light abstraction at top of PHP OOP stuctures such as namespaces, classes, class properties, methods and etc. In addition to that it provides FileElement used generally to write generated class into specified filename. Reactor elements will be described based on examples in this section.

## Generation
Before we will start generating our classes, their methods and properties let's shortly review FileElement as this is needed part to write generated code into file. FileElement has only one important dependency we have to provide - `FilesInterface`. All given examples uses service/controller methods.

```php
public function index()
{
    //We are going to create file element with primary
    //file namespace "MyNamespace"
    $file = new FileElement($this->files, 'MyNamespace');

    //We can also define namespace of primary namespace uses
    $file->addUse(Core::class);

    //Or use simplified method
    $file->setUses([Core::class]);

    //To add new class into file we can use method addClass
    $file->addClass( new ClassElement('MyClass'));
    
    //If you want to see what elements already located into 
    //file element use following methods
    dump($file->getUses());
    dump($file->getElements());

    //We can also set file header comment, reactor comments
    //can be specified in a form of string or array of lines
    $file->setComment([
        'This file was generated automatically.',
        'See components/reactor.'
    ]);

    //To write file to disk we have to only specify it's location,
    //mode and request to automatically create needed directory
    //Most of generated classes will be readonly from web user
    $file->render(
        directory('application') . '/classes/MyNamespace/MyClass.php',
        FilesInterface::READONLY,
        true
    );
}
```

Generated file will be located in application/classes/MyNamespace/MyClass.php file and will look like:

```php
<?php
/**
 * This file was generated automatically.
 * See components/reactor.
 */
namespace MyNamespace;
use Spiral\Core\Core;

class MyClass
{
}
```

## ClassElement
The most important reactor element which aggreages most of functionality located in ClassElement class. We can use such element to describe needed structure. Let's try to modify previous example first so we can easily add functionality to our class:

```php
public function index()
{
    //We are going to create file element with primary
    //file namespace "MyNamespace"
    $file = new FileElement($this->files, 'MyNamespace');

    $class = new ClassElement('MyClass');

    //To add new class into file we can use method addClass
    $file->addClass($class);

    //We can also set file header comment, reactor comments
    //can be specified in a form of string or array of lines
    $file->setComment([
        'This file was generated automatically.',
        'See components/reactor.'
    ]);

    //To write file to disk we have to only specify it's location,
    //mode and request to automatically create needed directory
    //Most of generated classes will be readonly from web user
    $file->render(
        directory('application') . '/classes/MyNamespace/MyClass.php',
        FilesInterface::READONLY,
        true
    );
}
```

All future examples will skip code realted to FileElement and writing class code into filesystem.

### Parent and interfaces
First of all we would like to define few class interfaces and it's parent. Do not forget to add interfaces and parent class name into file uses. In addtion to that let's set class doc comment value.

```php
$file->setUses([Component::class, \Countable::class]);
$class->setExtends('Component')->addInterface('Countable');

//Ad before we can read what parent and interfaces used by class
dump($class->getExtends());
dump($class->getInterfaces());
```

So far generated code will look like that:

```php
<?php
/**
 * This file was generated automatically.
 * See components/reactor.
 */
namespace MyNamespace;
use Spiral\Core\Component;
use Countable;

/**
 * This is our class.
 */
class MyClass extends Component  implements Countable
{
}
```

In a future examples i will skip methods dedicated to read sub elements and properties to make it ligher, simple check element getters.

### Constants and properties
Next we can add few constants and properties to our class.

```php
$class->setConstant('SOME_VALUE', 'ABC', ['This is my constant.']);

//Property represented by ProperyElement
$property = $class->property('myProperty', ['This is my static property.']);

$property->setStatic(true)->setAccess(PropertyElement::ACCESS_PRIVATE);

//We can also set default property value
$property->setDefault(true, [
    'a' => 1,
    'b' => 2
]);

//In addition let's add non static public property
$class->property(
    'nonStatic',
    ['Non static property.']
)->setAccess(PropertyElement::ACCESS_PUBLIC)->setDefault(true, null);
```

Resulted code:

```php
<?php
/**
 * This file was generated automatically.
 * See components/reactor.
 */
namespace MyNamespace;
use Spiral\Core\Component;
use Countable;

/**
 * This is our class.
 */
class MyClass extends Component  implements Countable
{
    /**
     * This is my constant.
     */
    const SOME_VALUE = 'ABC';

    /**
     * This is my static property.
     */
    private static $myProperty = [
        'a' => 1,
        'b' => 2
    ];

    /**
     * Non static property.
     */
    public $nonStatic = NULL;
}
```

### Methods
The most imporant part of any generated class is it's methods, every method are represented by MethodElements and ParameterElement.

```php
//Such method will be created with empty body
$class->method('count', ['This is count method.']);

//Let's work with "doSomething" method closer
$doMethod = $class->method('doSomething');

//First of all we can set method top comment (one extra line).
$doMethod->setComment(['This is do method.', '']);

//Not we can define method parameter by it's name and type,
//method will automatically generate section in doc comment
$doMethod->parameter('name', 'string');

//If we wish to work with parameter closer we can get instance
//of ParameterElement
$parameter = $doMethod->parameter('entity', 'DataEntity');

//We declared our parameter type as specified class name,
//let's add such class into file uses
$file->addUse(DataEntity::class);

//Next we can force parameter type in it's declaration (not comment)
$parameter->setType('DataEntity');

//We can also specify default parameter value
$parameter->setOptional(true, null);
```

Generated code:

```php
<?php
/**
 * This file was generated automatically.
 * See components/reactor.
 */
namespace MyNamespace;
use Spiral\Core\Component;
use Countable;
use Spiral\Models\DataEntity;

/**
 * This is our class.
 */
class MyClass extends Component  implements Countable
{
    /**
     * This is my constant.
     */
    const SOME_VALUE = 'ABC';

    /**
     * This is my static property.
     */
    private static $myProperty = [
        'a' => 1,
        'b' => 2
    ];

    /**
     * Non static property.
     */
    public $nonStatic = NULL;

    /**
     * This is count method.
     */
    public function count()
    {
    }

    /**
     * This is do method.
     * 
     * @param string $name
     * @param DataEntity $entity
     */
    public function doSomething($name, DataEntity $entity = NULL)
    {
    }
}
```

> You can also mark method as static, change it's access level and mark some parameters as passed by reference.

### Methods source
The most imporant part of MethodElement is ability to provide it's source code lines, let's write some code for our method.

```php
//First of all we want to add return value to our comment,
//let's use second argument to specify that we appending lines
$doMethod->setComment([
    '@return string'
], true);

//Default indentation is 4 tabs, source must be provides from
//first indentation level
$doMethod->setSource([
    'if (!empty($entity)) {',
    '    //This is pretty weird code',
    '    return get_class($entity);',
    '}',
    '',
    'return $name;'
]);
```

Our resulted class will look like:

```php
<?php
/**
 * This file was generated automatically.
 * See components/reactor.
 */
namespace MyNamespace;
use Spiral\Core\Component;
use Countable;
use Spiral\Models\DataEntity;

/**
 * This is our class.
 */
class MyClass extends Component  implements Countable
{
    /**
     * This is my constant.
     */
    const SOME_VALUE = 'ABC';

    /**
     * This is my static property.
     */
    private static $myProperty = [
        'a' => 1,
        'b' => 2
    ];

    /**
     * Non static property.
     */
    public $nonStatic = NULL;

    /**
     * This is count method.
     */
    public function count()
    {
    }

    /**
     * This is do method.
     * 
     * @param string $name
     * @param DataEntity $entity
     * @return string
     */
    public function doSomething($name, DataEntity $entity = NULL)
    {
        if (!empty($entity)) {
            //This is pretty weird code
            return get_class($entity);
        }
        
        return $name;
    }
}
```

Our resulted code needed to generate such class will be pretty huge but easly to modularize:

```php
public function index()
{
    //We are going to create file element with primary
    //file namespace "MyNamespace"
    $file = new FileElement($this->files, 'MyNamespace');

    $class = new ClassElement('MyClass');
    $class->setComment(['This is our class.']);

    $file->setUses([Component::class, \Countable::class]);
    $class->setExtends('Component')->addInterface('Countable');
    
    $class->setConstant('SOME_VALUE', 'ABC', ['This is my constant.']);

    //Property represented by ProperyElement
    $property = $class->property('myProperty', ['This is my static property.']);

    $property->setStatic(true)->setAccess(PropertyElement::ACCESS_PRIVATE);

    //We can also set default property value
    $property->setDefault(true, [
        'a' => 1,
        'b' => 2
    ]);

    //In addition let's add non static public property
    $class->property(
        'nonStatic',
        ['Non static property.']
    )->setAccess(PropertyElement::ACCESS_PUBLIC)->setDefault(true, null);

    //Such method will be created with empty body
    $class->method('count', ['This is count method.']);

    //Let's work with "doSomething" method closer
    $doMethod = $class->method('doSomething');

    //First of all we can set method top comment (one extra line).
    $doMethod->setComment(['This is do method.', '']);

    //Not we can define method parameter by it's name and type,
    //method will automatically generate section in doc comment
    $doMethod->parameter('name', 'string');

    //If we wish to work with parameter closer we can get instance
    //of ParameterElement
    $parameter = $doMethod->parameter('entity', 'DataEntity');

    //We declared our parameter type as specified class name,
    //let's add such class into file uses
    $file->addUse(DataEntity::class);

    //Next we can force parameter type in it's declaration (not comment)
    $parameter->setType('DataEntity');

    //We can also specify default parameter value
    $parameter->setOptional(true, null);

    //First of all we want to add return value to our comment,
    //let's use second argument to specify that we appending lines
    $doMethod->setComment([
        '@return string'
    ], true);

    //Default indentation is 4 tabs, source must be provides from
    //first indentation level
    $doMethod->setSource([
        'if (!empty($entity)) {',
        '    //This is pretty weird code',
        '    return get_class($entity);',
        '}',
        '',
        'return $name;'
    ]);

    //To add new class into file we can use method addClass
    $file->addClass($class);

    //We can also set file header comment, reactor comments
    //can be specified in a form of string or array of lines
    $file->setComment([
        'This file was generated automatically.',
        'See components/reactor.'
    ]);

    //To write file to disk we have to only specify it's location,
    //mode and request to automatically create needed directory
    //Most of generated classes will be readonly from web user
    $file->render(
        directory('application') . '/classes/MyNamespace/MyClass.php',
        FilesInterface::READONLY,
        true
    );
}
```

## Generators and Scaffoling
In addition to low level element manipulation, spiral provides set of pre-created code generator and command to be used for application scaffolding:

| Command           | Generator Class                               | Description                   |
| ---               | ---                                           | ---                           |
| create:command    | Spiral\Reactor\Generators\CommandGenerator    | Generate new command.         |
| create:controller | Spiral\Reactor\Generators\ControllerGenerator | Generate new controller.      |
| create:document   | Spiral\Reactor\Generators\DocumentGenerator   |  Generate new ODM document.   |
| create:middleware | Spiral\Reactor\Generators\MiddlewareGenerator | Generate new http middleware. |
| create:migration  | Spiral\Reactor\Generators\MigrationGenerator  | Generate new migration.       |
| create:record     | Spiral\Reactor\Generators\RecordGenerator     | Generate new ORM Record.      |
| create:request    | Spiral\Reactor\Generators\RequestGenerator    | Generate new request filter.  |
| create:service    | Spiral\Reactor\Generators\ServiceGenerator    | Generate new service.         |

> You can check source of generators and help for listed commands to integrate scaffoling into your application or module. Most of generators can be extended to add new functionality without any problem.
