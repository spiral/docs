# Application Memory
One of the notable spiral components which deserves its own section is the application memory component or `HippocampusInterface`. The memory interface is responsible
for storing component cache information into "permanent" application storage. The default spiral implementation will generate php files in [runtime directory](/application/directories.md) to store provided data.

The general idea of memory is to speed up application bootstrapping and move some runtime operations into the backgroud. The memory component is used to store the configuration cache,
orm and odm schema, loadmap, console commands and tokenizer cache; it can also be used to cache compiled routes and etc. Application memory is very similar to the cache component however it must never be used to store any data related to a client request.

So to simply state the purpose of the `HippocampusInterface` let's say it's very expensive to store information in, and very quick to retrieve it back.

```php
interface HippocampusInterface
{
    /**
     * Read data from long memory cache. Must return exacts same value as saved or null.
     *
     * @param string $section  Non case sensitive.
     * @param string $location Specific memory location.
     * @return string|array|null
     */
    public function loadData($section, $location = null);

    /**
     * Put data to long memory cache. No inner references or closures are allowed.
     *
     * @param string       $section  Non case sensitive.
     * @param string|array $data
     * @param string       $location Specific memory location.
     */
    public function saveData($section, $data, $location = null);

    /**
     * Get all memory sections belongs to given memory location (default location to be used if
     * none specified).
     *
     * @param string $location
     * @return array
     */
    public function getSections($location = null);
}
```

## Memory usage example
Let's view an example of a service which needs to index available classes and generate set of operations based on it:

```php
abstract class Operation 
{
    /**
     * Execute some operation.
     */
    abstract public function perform($request);
}

class OperationService
{
    /**
     * List of operation associated with thier class.
     */
    protected $operations = [];

    /**
     * OperationService constructor.
     *
     * @param HippocampusInterface  $memory
     * @param ClassLocatorInterface $locator
     */
    public function __construct(HippocampusInterface $memory, ClassLocatorInterface $locator)
    {
        $this->operations = $memory->loadData('operations');

        if (is_null($this->operations)) {
            $this->operations = $this->locateOperations($locator);
        }

        //We now can store data into long time memory
        $memory->saveData('operations', $this->operations);
    }

    /**
     * @param string $operation
     * @param mixed  $request
     */
    public function run($operation, $request)
    {
        //Perform operation based on $operations property
    }

    /**
     * @param ClassLocatorInterface $locator
     * @return array
     */
    protected function locateOperations(ClassLocatorInterface $locator)
    {
        //Generate list of available operations via scanning every available class
    }
}
```

> You can only store arrays and scalar values in long term memory.

`HippocampusInterface` is implemented in spiral bundle using `Memory` class, which lets you access it's functions using short 'memory' binding in your services or controllers.

```php
public function doSomething()
{
    dump($this->memory->loadData('something'));
}
```

You can implement your own version of `HippocampusInterface` using APC, XCache or even Memcache. 

## Embedding Memory into Components
Before you will embed `HippocampusInterface` into your component or service:
* Do not expect that stored data will always be in memory, it might dissapear at any moment.
* Do not store any data related to user request, action or information. Memory is only for logic caching
