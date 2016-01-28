# Tokenizer
One of the important parts of spiral framework located in Tokenizer component. Such set of classes and
interfaces used to perform soft static analysis of your codebase, including location of specific
class declarations, method and function invocations and php blocks isolation.

## ClassLocatorInterface
ClassLocatorInterface is very simple abstraction used by ORM, ODM and Console components to locate
existed declaration of classes:

```php
interface ClassLocatorInterface
{
    /**
     * Index all available files and generate list of found classes with their names and filenames.
     * Unreachable classes or files with conflicts must be skipped. This is SLOW method, should be
     * used only for static analysis.
     *
     * Output format:
     * $result['CLASS_NAME'] = [
     *      'class'    => 'CLASS_NAME',
     *      'filename' => 'FILENAME',
     *      'abstract' => 'ABSTRACT_BOOL'
     * ]
     *
     * @param mixed $target  Class, interface or trait parent. By default - null (all classes).
     *                       Parent (class) will also be included to classes list as one of
     *                       results.
     * @return array
     */
    public function getClasses($target = null);
}
```

Classical usage of such interface might look like:

```php
public function indexAction(ClassLocatorInterface $locator)
{
    //Every declared controller
    dump($locator->getClasses(ControllerInterface::class));
    
    //Every declared ORM RecordEntity class
    dump($locator->getClasses(RecordEntity::class));
}
```

By default, class locator works with `ReflectionFile` instance provided by `TokenizerInterface` and
scans every registred tokenization directory (see below). If you want to decouple your project from
spiral, simply create your own implementation of class locator basen on config, static array or
other location method.

## File Reflection
As mentioned in previous section, default implementation of ClassLocator uses ReflectionFile class
as it's source of declarations. In some cases you might want to get access to ReflectionFile
inside your component or module. This can be achived via `TokenizerInterface` method `fileReflection`:

```php
public function indexAction(TokenizerInterface $tokenizer)
{
    dump($tokenizer->fileReflection(__FILE__)->getClasses());
}
```

> ReflectionFile might help you to fetch class, interface, trait or function declaration from your file.

## InvocationLocatorInterface
Separatelly from ClassLocator, you might use InvocationLocator. Such interface is based on a ReflectionFile
method `getInvocations` (see previous section) and might help your to perform simple analysis of your code
to detect where some method were used.

For example, we can use our locator to find every invoked internalization function:

```php
public function indexAction(InvocationLocatorInterface $locator)
{
    dump($locator->getInvocations(new \ReflectionFunction('l')));
    l('hello world!');
}
```

> At this moment InvocationLocator based on php tokenization and not AST, this limits locations to function,
static or $this calls only. Class is capable of locating trait methods (say, logger, fire and etc).

## Tokenizer configuration
Previous sections explains how we can locate our classes, models and invocations in our application source files.
Hovewer, tokenizer will only count some specific files to be belong to your application based on list of directories
and exclude patterns (see Symfony\Finder) defined in tokenizer configuration:

```php
return [
    /*
     * Tokenizer will be performing class and invocation lookup in a following directories. Less
     * directories - faster Tokenizer will work.
     */
    'directories' => [
        directory('application'),
        directory('libraries') . '/spiral/framework',
        directory('libraries') . '/spiral/components',
        directory('libraries') . '/spiral/scaffolder',
        /*{{directories}}*/
    ],
    /*
     * Such paths are excluded from tokenization. You can use format compatible with Symfony Finder.
     */
    'exclude'     => [
        'vendor',
        /*{{exclude}}*/
    ]
];
```

Since location or Commands, Records and Documents are based on spiral tokenizer you might need to register
your module path in tokenizer config in order to let system find your declarations, this can be achieved
by adding following code to your module class:

```php
/**
 * {@inheritdoc}
 */
public function register(RegistratorInterface $registrator)
{
    $registrator->configure('tokenizer', 'directories', 'vendor/package', [
        "directory('libraries') . 'vendor/package'"
    ]);
}
```

## PHP blocks isolator
PHP blocks isolation functionality located in `Spiral\Tokenizer\Isolator` class, generally speaking
this implementation is only responsible for locating, isolating and replacing php block in a given 
string. This implementation used a lot in view processors to ensure that no php code will be affected
by template processing.

```php
$source = '<?php echo 1, 2 ?> hello world <?= $var ?>';

$isolator = new Isolator();

$isolated = $isolator->isolatePHP($source);
dump($isolated); //-php-random-block- hello world -php-random-block-

$restored = $isolator->repairPHP($isolated);
dump($restored); 
```
