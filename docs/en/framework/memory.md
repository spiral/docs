# Framework - Static Memory
Framework (component `spiral/boot`) provides a convenient interface to store some computation data shared between processes.  

> Current implementation of shared memory stores data in physical files with the help of OpCache. Future implementations will
move data storage to RoadRunner or shared PHP extension with SHM, do not couple your codebase to physical files.  

## MemoryInterface
Use interface `Spiral\Boot\MemoryInterface` to store computation results:

```php
/**
 * Long memory cache. Use this storage to remember results of your calculations, do not store user
 * or non-static data in here (!).
 */
interface MemoryInterface
{
    /**
     * Read data from long memory cache. Must return exacts same value as saved or null. Current
     * convention allows to store serializable (var_export-able) data.
     *
     * @param string $section Non case sensitive.
     * @return string|array|null
     */
    public function loadData(string $section);

    /**
     * Put data to long memory cache. No inner references or closures are allowed. Current
     * convention allows to store serializable (var_export-able) data.
     *
     * @param string       $section Non case sensitive.
     * @param string|array $data
     */
    public function saveData(string $section, $data);
}
```

## Use Cases
The general idea of memory is to speed up an application by caching the execution result of some functionality. The memory component used to store the configuration cache, ORM and ODM schemas, console commands list and tokenizer cache;
it can also be used to cache compiled routes, etc.

 > Application memory must never be used to store user data.

## Practical Example
Let's view an example of a service used to analyze available classes to compute some behavior (operations):

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
     * List of operation associated with their class.
     */
    protected $operations = [];

    /**
     * OperationService constructor.
     *
     * @param MemoryInterface  $memory
     * @param ClassesInterface $classes
     */
    public function __construct(MemoryInterface $memory, ClassesInterface $classes)
    {
        $this->operations = $memory->loadData('operations');

        if (is_null($this->operations)) {
            $this->operations = $this->locateOperations($classes); // slow operation
            $memory->saveData('operations', $this->operations);
        }      
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
     * @param ClassesInterface $locator
     * @return array
     */
    protected function locateOperations(ClassesInterface $classes)
    {
        //Generate list of available operations via scanning every available class
    }
}
```

> You can currently only store arrays or scalar values in memory.

You can implement your version of `Spiral\Boot\MemoryInterface` using APC, XCache, DHT on RoadRunner, Redis, or even Memcache.

Before you embed `Spiral\Boot\MemoryInterface` into your component or service:
* Do not store any data related to a user request, action or information. Memory is only for logic caching
* Assume memory can disappear at any moment
* `saveData` is thread-safe but slows down with higher concurrency
* `saveData` is more expensive than `loadData`, make sure not to store anything in memory during application runtime
* bootloaders and commands are the best places to use memory
