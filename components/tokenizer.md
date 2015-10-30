# Tokenizer
One of the most important spiral features are located in Tokenizer component, such component are responsible for locating classes and their filenames based on provided parent, interface or even trait. Such functionality widely used in ORM, ODM, Modules, Console Commands and other parts of framework to simplify developer life and locate 
required code pieces automatically.

## TokenizerInterface
Let's check `Spiral\Tokenizer\TokenizerInterface` to see what methods available for us:

```php
interface TokenizerInterface
{
    /**
     * Token array constants.
     */
    const TYPE = 0;
    const CODE = 1;
    const LINE = 2;

    /**
     * Fetch PHP tokens for specified filename. Usually links to token_get_all() function. Every
     * token MUST be converted into array.
     *
     * @param string $filename
     * @return array
     */
    public function fetchTokens($filename);

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
     * @param mixed $parent  Class, interface or trait parent. By default - null (all classes).
     *                       Parent (class) will also be included to classes list as one of
     *                       results.
     * @return array
     */
    public function getClasses($parent = null);
}
```

> In spiral such interface implemented by class `Spiral\Tokenizer\Tokenizer` which provides few additional features including ability to find only classes from specific namespace and specific postfix in `getClasses` method.

As you see the most important part of Tokenizer component located in getClasses method. Let's try to view some examples (as in other cases we can use short core binding "tokenizer" in our Controllers and Services):

```php
protected function indexAction()
{
    //Find every controller class
    dump($this->tokenizer->getClasses(ControllerInterface::class));

    //Find every Record class
    dump($this->tokenizer->getClasses(Record::class));

    //Every class which extends current class
    dump($this->tokenizer->getClasses($this));

    //Every class which uses LoggerTrait and located in Spiral namespace
    dump($this->tokenizer->getClasses(LoggerTrait::class, 'Spiral'));
}
```

> You have to rememeber that Tokenizer `getClasses` is VERY slow method, you should never use it in runtime.

## Default Tokenizer Implementation
Default TokenizerInterface implementation provides few additional features which can be useful for you. First of all, we can always get every used trait for given class (including traits used by parents):

```php
protected function indexAction()
{
    dump($this->tokenizer->getTraits(self::class));
}
```

Second feature must be treated as experimental (spiral might migrate to PHPParsed one day), and provides you ablity to get ReflectionFile instance which will provide
information about classes, interfaces, traits and function declared in given file and, in addition, allow you to located function calls (this feature used by Translator to index your i18n function usages):

```php
protected function indexAction()
{
    $reflection = $this->tokenizer->fileReflection(__FILE__);

    dump($reflection->getClasses());

    //Will locate every "dump" function, "self::something" call and it's arguments
    dump($reflection->getCalls());

    self::something($reflection, "string");
}

public static function something()
{

}
```

> Tokenizer can clearly locate only function and static method calls.
