# Application Tests
Testing your code is essential in order to develop reliable application. Besides classic unit tests for your services and/or controllers you are able to test your application as whole using isolated environment.

## BaseTest
Locate this file in `tests` directory of your application, by default it configured to work with default `.env`. Test built in a way so your application will be constructed and destroyed completely for each test case:

```php
abstract class BaseTest extends TestCase
{
    /**
     * @var \App
     */
    protected $app;

    public function setUp()
    {
        $root = dirname(__DIR__) . '/';

        $app = $this->app = \App::init([
            'root'        => $root,
            'libraries'   => $root . 'vendor/',
            'application' => $root . 'app/',
        ], null, null, false);

        //Monolog love to write to CLI when no handler set
        $this->app->logs->debugHandler(new NullHandler());
    }

    /**
     * This method performs full destroy of spiral environment.
     */
    public function tearDown()
    {
        if (class_exists('Mockery')) {
            //Mockery defines it's own static container to be destructed
            \Mockery::close();
        }

        //Forcing destruction
        $this->app = null;

        gc_collect_cycles();
        clearstatcache();
    }
}
```

You can pass custom instance of `EnvironmentInterface` in order to define separate database credentials and values. 

## Testing Http Methods
Use `HttpTest` with ability to emulate user requests, cookies and sessions:

```php
class IndexTest extends HttpTest
{
    public function testSeeWelcome()
    {
        $response = $this->get('/');

        $this->assertSame(200, $response->getStatusCode());
        $this->assertContains('Welcome to Spiral Framework', (string)$response->getBody());
        $this->assertContains('welcome.dark.php', (string)$response->getBody());
    }
}
```

## Preparing environment
If you wish to test your application on clean database in each of your test cases simply call needed setup commands in your setup method:

```php
public function setUp()
{
    $root = dirname(__DIR__) . '/';

    $app = $this->app = \App::init([
        'root'        => $root,
        'libraries'   => $root . 'vendor/',
        'application' => $root . 'app/',
    ], null, null, false);

    //Monolog love to write to CLI when no handler set
    $this->app->logs->debugHandler(new NullHandler());
    
    //Scaffold databases
    $this->app->console->run('orm:schema', ['--alter' => true]);
}
```

Such approach can be beneficial when you testing code in relation to real database, though it will drastically increase test suite duration. Consider switching to memory based databases (for example SQLite) in order to speed it up.