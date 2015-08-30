# Application Memory
One of the notable spiral components which deserves it's own section is the application memory component or `HippocampusInterface`. Memory interface responsible
for storing component cache information into "permanent", enviroment specific storage. The default spiral implementation will generate php file in [runtime
directory] (/application/directories.md) to store provided data.

General idea of memory is to speed up application bootstrapping and move some runtime operations into the backgroud, memory used to store configuration cache,
orm and odm schema, loadmap, console commands and tokenizer cache; it can also be used to cache compiled routes and etc. Application memory are very similar to cache component however it must never be used to store any data related to client request.

So to simply state the purpose of the `HippocampusInterface` let's say it's very expensive to store information in, and very quick to retrieve it back.

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

    public function __construct(HippocampusInterface $memory, TokenizerInterface $tokenizer)
    {
        if(is_null($this->operations = $memory->loadData('operations')) {
            $this->operations = $this->locateOperations($tokenizer);
            
            //We now can store data into long time memory
            $memory->saveData('operations', $this->operations); 
        }
    }

    public function run($operation, $request)
    {
        //Perform operation based on $operations property
    }
    
    protected function locateOperations(TokenizerInterface $tokenizer)
    {
        //Generate list of available operations via scanning every available class
    }
}
```

> You can only store arrays and scalar values in long term memory.

`HippocampusInterface` is implemented in spiral bundle using `Core` and `Application` classes, which lets you access it's functions using short 'core' binding in your services or controllers.

```php
public function doSomething()
{
    dump($this->core->loadData('something'));
}
```

You can implement your own version of `HippocampusInterface` using APC, XCache or even Memcache. 

## Embedding Memory into Components
Before you will embed `HippocampusInterface` into your component or service you must remember 3 basic rules:
* Do not expect that stored data will always be in memory, it might dissapear at any moment.
* Do not store any data related to user request, action or information. Memory is only for logic caching.
* Do not feed this thing after midnight.
