# Reactor
Reactor component is not considered a part of Spiral components and exists only in the Spiral framework bundle. This component provides the ability to generate classes with any desired structure and functionality using logical definitions and avoiding the use of templates. The ability to do this provides a more flexible way to create scaffolding functionality for different framework parts.

> This component can be replaced by [Zend Code Generator](http://framework.zend.com/manual/current/en/modules/zend.code.generator.examples.html).

Reactor can be separated into two general pieces: elements and generators.

## Reactor Elements
Reactor element provides light abstraction at the top of any PHP OOP stucture such as namespaces, classes, class properties, methods etc. In addition, it gives FileElement which is generally used to write generated class into specified filename. Reactor elements will be defined based on the examples in this section.

## Generation
Before we review how classes, their methods and properties are generated, let's take a look at FileElement as this is neccessary in order to write the generated code into a file. FileElement has only one important element we must provide - `FilesInterface`. All given examples use service/controller methods.

```php
public function index()
{
    //We are going to create a file element with primary
    //file namespace "MyNamespace"
    $file = new FileElement($this->files, 'MyNamespace');

    //We can also define namespace that the primary namespace uses
    $file->addUse(Core::class);

    //Or use simplified method
    $file->setUses([Core::class]);

    //To add new class into file we can use method addClass
    $file->addClass( new ClassElement('MyClass'));
    
    //If you want to see what elements are already located into 
    //file element use the following methods
    dump($file->getUses());
    dump($file->getElements());

    //We can also set a file header comment. Reactor comments
    //can be defined by a form of string or array of lines
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

The generated file will be located in application/classes/MyNamespace/MyClass.php file and will look like this:

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
The most important reactor element which aggregates much of its functionality is located in ClassElement class. We can use this element to describe the structure we need. Let's try to modify the previous example first so we can easily add this functionality to our class:

```php
public function index()
{
    //We are going to create file element with primary
    //file namespace "MyNamespace"
    $file = new FileElement($this->files, 'MyNamespace');

    $class = new ClassElement('MyClass');

    //To add this new class into file, we can use method addClass
    $file->addClass($class);

    //We can also set a file header comment. Reactor comments
    //can be specified by a form of string or array of lines
    $file->setComment([
        'This file was generated automatically.',
        'See components/reactor.'
    ]);

    //To write file to disk, we have to specify it's location,
    //mode and request to automatically create the needed directory
    //Most of generated classes will be readonly from web user
    $file->render(
        directory('application') . '/classes/MyNamespace/MyClass.php',
        FilesInterface::READONLY,
        true
    );
}
```

All future examples will skip the code related to FileElement and will instead write the class code into filesystem.

### Parent and interfaces
First, we need to define a few class interfaces and their parent. Do not forget to add interfaces and parent class name into file uses. Additionally, let's set the class doc comment value.

```php
$file->setUses([Component::class, \Countable::class]);
$class->setExtends('Component')->addInterface('Countable');

//As was the case before, we can read what parent and interfaces are used by class
dump($class->getExtends());
dump($class->getInterfaces());
```

So far your generated code will look like this:

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

In future releases, contributors to Spiral will skip methods dedicated to read sub elements and properties to make it faster and easier to check element getters.

### Constants and properties
Next we can add a few constants and properties to our class.

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

//In addition, let's add some non static public property
$class->property(
    'nonStatic',
    ['Non static property.']
)->setAccess(PropertyElement::ACCESS_PUBLIC)->setDefault(true, null);
```

Resulting code:

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
Methods are the most important part of any generated class.  Every method is represented by MethodElements and ParameterElement.

```php
//This method will be created with an empty body
$class->method('count', ['This is count method.']);

//Let's examine "doSomething" method closer
$doMethod = $class->method('doSomething');

//First of all, we can set the method top comment (one extra line).
$doMethod->setComment(['This is do method.', '']);

//Now we can define method parameter by it's name and type,
//method will automatically generate section in doc comment
$doMethod->parameter('name', 'string');

//If we wish to work with parameter closer we can get instance
//of ParameterElement
$parameter = $doMethod->parameter('entity', 'DataEntity');

//We declared our parameter type as a specified class name,
//let's add this class into file uses
$file->addUse(DataEntity::class);

//Next, we can force parameter type in it's declaration (not comment)
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

> You can also mark your method as static, change it's access level and/or mark some parameters as passed by reference.

### Methods source
The most imporant part of MethodElement is the ability to provide it's source code lines. Let's write some code for our method.

```php
//First of all, we want to add return value to our comment.
//Lets use a second argument to specify that we are appending lines
$doMethod->setComment([
    '@return string'
], true);

//Default indentation is 4 tabs. Source must be provided from
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

Our generated class will look like this:

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

The resulting code which was needed to generate such a class will be fairly large but easy to make modular:

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

    //In addition, let's add in non static public property
    $class->property(
        'nonStatic',
        ['Non static property.']
    )->setAccess(PropertyElement::ACCESS_PUBLIC)->setDefault(true, null);

    //Such a method will be created with an empty body
    $class->method('count', ['This is count method.']);

    //Let's work with the "doSomething" method more closely
    $doMethod = $class->method('doSomething');

    //First of all, we can set the method top comment (one extra line).
    $doMethod->setComment(['This is do method.', '']);

    //Now we can define the method parameter by it's name and type. 
    //The method willautomatically generate a section in doc comment
    $doMethod->parameter('name', 'string');

    //If we want to work with parameter more closely, we can get instance
    //of ParameterElement
    $parameter = $doMethod->parameter('entity', 'DataEntity');

    //We declared our parameter type as a specified class name.
    //Let's add this class into file uses
    $file->addUse(DataEntity::class);

    //Next we can force the parameter type in it's declaration (not comment)
    $parameter->setType('DataEntity');

    //We can also specify the default parameter value
    $parameter->setOptional(true, null);

    //First, we want to add the return value to our comment.
    //Lets use the second argument to specify that we appending lines
    $doMethod->setComment([
        '@return string'
    ], true);

    //Default indentation is 4 tabs. The source must be provided from the
    //first indentation level
    $doMethod->setSource([
        'if (!empty($entity)) {',
        '    //This is pretty strange code',
        '    return get_class($entity);',
        '}',
        '',
        'return $name;'
    ]);

    //To add a new class into the file, we can use the method addClass
    $file->addClass($class);

    //We can also set a file header comment. Reactor comments
    //can be specified by a form of string or array of lines
    $file->setComment([
        'This file was generated automatically.',
        'See components/reactor.'
    ]);

    //To write the file to disk, we only have to specify it's location,
    //mode and request to automatically create the neccessary directory.
    //Many generated classes will be readonly from web user
    $file->render(
        directory('application') . '/classes/MyNamespace/MyClass.php',
        FilesInterface::READONLY,
        true
    );
}
```

## Generators and Scaffoling
In addition to low level element manipulation, Spiral provides a set of pre-created code generators and commands that can used for application scaffolding:

| Command           | Generator Class                               | Description                   |
| ---               | ---                                           | ---                           |
| create:command    | Spiral\Reactor\Generators\CommandGenerator    | Generate new command.         |
| create:controller | Spiral\Reactor\Generators\ControllerGenerator | Generate new controller.      |
| create:document   | Spiral\Reactor\Generators\DocumentGenerator   | Generate new ODM Document.    |
| create:middleware | Spiral\Reactor\Generators\MiddlewareGenerator | Generate new http middleware. |
| create:migration  | Spiral\Reactor\Generators\MigrationGenerator  | Generate new migration.       |
| create:record     | Spiral\Reactor\Generators\RecordGenerator     | Generate new ORM Record.      |
| create:request    | Spiral\Reactor\Generators\RequestGenerator    | Generate new request filter.  |
| create:service    | Spiral\Reactor\Generators\ServiceGenerator    | Generate new service.         |

> You can check the source of generators and support for listed commands to integrate scaffolding into your application or module. Most generators can be used to add new functionality without a problem.
