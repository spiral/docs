# Debug
Spiral provides embedded debug component with set of classes useful in developing and debugging your application. Debug does mean to replace existed debugging tools like XDebug and create to only simplify your application development and profiling in remote enviroments.

## Loggers
Spiral Logger class built at top of PSR3 standard and compatible with it. You can simply replace default application logger using Monolog classes and etc.
Spiral loggers aggregate log messages using channel name, in most of cases (when LoggerTrait was used) channel name will be the same as parent class.

If you wish to access logger in your code (for example controller action) you can simply declare dependency:

```php
public function index(Logger $logger)
{
    $logger->alert('abc!');
    
    //Loggers support message interpolation
    $logger->warning('message {name}!', ['name' => 'value']);
}
```

> This code will create instance of Logger associated with global channel "@global". Spiral 

### Log Handlers
Spiral provides you simplistic way to create handlers to store or send specific log messages, you can add your handler to Logger using setHandler method:

```php
public function index(Logger $logger)
{
    $logger->setHandler(Logger::ERROR, new MyHandler());
}
```

Your handler must only define __invoke method with following arguments:

```php
/**
 * Handle log message.
 *
 * @param int    $level
 * @param string $message
 * @param array  $context
 */
public function __invoke($level, $message, array $context = []);
```

If you wish your handler to log every log message (every level) you can assing it to `Logger::ALL`:

```php
public function index(Logger $logger)
{
    $logger->setHandler(Logger::ALL, function ($level, $messsage, array $context) {
        dump($level);
        dump($message);
    });
}
```

> Message will come pre-interpolated into handler method.

Every logger instance requested via dependency injection will receive instance of Debug component, such instance provides ability to pre-configure some 
log handlers using debug configuraion:

```php
'loggers'       => [
    Logger::GLOBAL_CHANNEL  => [
        Logger::ALL   => [
            'class'    => FileHandler::class,
            'filename' => directory('runtime') . '/logging/global.log'
        ]
    ],
    Debugger::class       => [
        Logger::ERROR => [
            'class'    => FileHandler::class,
            'filename' => directory('runtime') . '/logging/errors.log'
        ],
        Logger::ALL   => [
            'class'    => FileHandler::class,
            'filename' => directory('runtime') . '/logging/debug.log'
        ]
    ],
    HttpDispatcher::class => [
        Logger::WARNING => [
            'class'    => FileHandler::class,
            'filename' => directory('runtime') . '/logging/httpErrors.log'
        ]
    ]
],
```

As you can see you are able to assing handler to specific channels and log levels, hovewer this way has some limitations - all used handlers must implement instance of `HandlerInterface` in order to be properly created and configured:

```php
interface HandlerInterface
{
    /**
     * HandlerInterface should only accept options from debug. Use depends method for additional
     * classes.
     *
     * @param array $options
     */
    public function __construct(array $options);

    /**
     * Handle log message.
     *
     * @param int    $level
     * @param string $message
     * @param array  $context
     */
    public function __invoke($level, $message, array $context = []);
}
```

Default file handler which you can see in configuration provides following set of options you can alter or leave default:

```php
/**
 * @var array
 */
protected $options = [
    'filename'      => '',                             //This option must be declared
    'filesize'      => 2097152,                        //Max log filesize is 2MB
    'mode'          => FilesInterface::RUNTIME,        //Mode
    'rotatePostfix' => '.old',                         //When log size exeeds max filesize it's will be rotated by addind this
                                                       //postfix to filename    
    'format'        => '{date}: [{level}] {message}',  //Messaged in log file will look like that
    'dateFormat'    => 'H:i:s d.m.Y'                   //Date format will be interpolated into line format
];
```

### Logger Trait
To simplify access to loggers spiral provides `LoggerTrait` which is compatible with `LoggerAwareInterface` but provides few additional features. Let's try to view example using some controller:

```php
class HomeController extends Controller
{
    use LoggerTrait;

    public function index()
    {
        $this->logger()->alert('test');
    }
}
```

As you can see LoggerTrait defined helper method for us `logger()` which will create insatnce of `LoggerInterface` of us on demand, the only argument will be provided to such logger is `name` which will be equal to class name (Controllers\HomeController).

> By default spiral binds `LoggerInterface` to `Logger`.

LoggerTrait has few abilities you might consider:
* by default Logger instance created by logger() method will be assigned to class statically, meaning every instance of our controller will be writing to same log
* there is additional variable in `setLogger` method - static (false by default), which provides you ability to set logger for instance or specific class

### Global log
Spiral `Logger` class provides one extra ability which can be very useful when working with `Profiler` module - global log. Global log can be turned on and off in debug configuration, when it's turned on every raised log message will be aggreated in `Debug` component and can be analyzed later.

Let's look into global log configuration section in our `debug.php` file:

```php
    'globalLogging' => [
        'enabled' => true,
        'maxSize' => 1000
    ],
```

> Database component can log every made query, `Profiler` module will not only display but format and highlight this queries.

## Benchmarks
Spiral provides simplistic trait which can help to record and profile different parts of your application, to use benchmarks we only need to connect BenchmarkTrait to your component or service (every Controller already have `BenchmarkTrait`).

```php
class TestService extends Service
{
    use BenchmarkTrait;

    public function method()
    {
        //Opening benchmark
        $benchmark = $this->benchmark('doing something');

        sleep(2);

        //Closing, benchmark will return elapsed seconds when closed
        dump($this->benchmark($benchmark));
    }
}
```

> Attention, benchmarking will work only if `Spiral\Debug\BenchmarkerInterface` defined in your application, Profiler module will define such class automatically when being initialized.

Benchmarking functionality may look not so useful, hovewer most critical part of spiral (queries, adapter creations, storage manipulations) are covered with them, as result you can use Profiler module and it's panel to view timeline of your application:

```php
public function index(Database $db)
{
    foreach ($db->getTables() as $table) {
        foreach ($table->schema()->getColumns() as $column) {
            dump($column);
        }
    }
}
```

![Profiler benchmarks](https://raw.githubusercontent.com/spiral/guide/master/resources/profiler.png)

## Dumps
One additional feature can help you to view content of objects or variables without launching XCache, for this feature we will need instance of `Dumper`:

```php
public function index(Dumper $dumper, MemcacheStore $memcacheStore)
{
    $dumper->dump($memcacheStore);
}
```

Following code will dump structure of $memcacheStore into output and will look like that:
![Dump](https://raw.githubusercontent.com/spiral/guide/master/resources/dumps.png)

You can also use short "dump" function in spiral environment (your probably notice it's in my examples):

```php
public function index(MemcacheStore $memcacheStore)
{
    dump($memcacheStore);
}
```

Spiral dumper fully support `__debugInfo` [method](http://php.net/manual/en/language.oop5.magic.php) which allows you to dump only important information. In addition to than you can hide certain fields of your objects (for example container property due it's dump can be pretty bit) by adding "@invisible" annotation.

```php
class TestService extends Service
{
    /**
     * Dumping this property will give us nothing, so we can hide it.
     *
     * @invisible
     * @var ContainerInterface
     */
    protected $container = null;
}
```

Dumper and dump method in addition to outputing content into active buffer can send dump info into different outputs:

```php
public function index(MemcacheStore $memcacheStore)
{
    //Output buffer
    dump($memcacheStore, Dumper::OUTPUT_ECHO);
    
    //Dump information in Spiral\Debug\Debugger log using print_r
    dump($memcacheStore, Dumper::OUTPUT_LOG);

    //Dump information in Spiral\Debug\Debugger log with nice formatting
    dump($memcacheStore, Dumper::OUTPUT_LOG_NICE);

    //Return dump as string
    $dump = dump($memcacheStore, Dumper::OUTPUT_RETURN);

}
```

## Profiler Module
You can achieve best development beformance by enabling `Profiler` middleware in your primary middleware chain located in `application/config/http.php` profiler will display loaded classes, aplication timeline using captured benchmarks and global log messages.
