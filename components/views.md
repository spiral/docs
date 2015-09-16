# Views
The Spiral view manager is a part of the framework bundle and not the components set. The default implementation includes support for multiple rendering engines (compiler + renderer), caching, cache dependencies and **view namespaces**.

## A few examples
Before we start to dig into some of the features and functionality of ViewManager, let's examine a few examples. First, we have to create a view file to be rendered. Like we see in many other frameworks, you can pass variables into this file. We are going to name our file "demo.php" and place it into "application/views" directory.

```php
This is a demo view file: <?= $value ?>
```

Next, we can render this file using `ViewsInterface` dependency or short binding "views" (we are going to work inside controller):

```php
public function index()
{
    return $this->views->render('demo', [
        'value' => mt_rand(0, 1000)
    ]);
}
```

If we open our action in the browser, we will see the rendered file with some random value in it. Every view file rendered using `ViewInterface` classes will have the default Spiral implementation of these interface located in `Spiral\Views\View`. This provides us with access to view object and container located within it. Let's update both the view file and our controller code.

Let's use the view container to access some service or class inside our view:

```php
This is a demo view file: <?= $value ?>

<?php
/**
 * @var \Spiral\Debug\Dumper $dumper
 */
$dumper = $this->container->get(\Spiral\Debug\Dumper::class);
$dumper->dump($value);
?>
```

In addition, we can use the 'get' method of our ViewManager to receive an instance of View and, if we wanted to, configure it before rendering:

```php
public function index()
{
    /**
     * @var View $view
     */
    $view = $this->views->get('demo');
    $view->set('value', mt_rand(0, 1000));

    //We can simply use "return $view;" due MiddlewarePipeline
    //this converts every response into a string
    return $view->render();
}
```

The code above is exactly the same as the short 'render' method. However it gives us the ability to manipulate the View class before rendering. Spiral provides us with the ability to register additional engines associated with a specified file extension and/or custom view rendered which can be very useful in some cases.

## View Namespaces
One of the core tenets of ViewManager is to aggregate and store view filenames into multiple locations by namespace. rendering/compilation phase ViewManager will walk though every namespace directory until view file is not found. View file is identified only by it's name. You can use an extension to locate what rendering/compilation engine to use. Let's look at our view configuration to check how the namespaces should defined:

```php
'namespaces'   => [
    'default'  => [
        directory("application") . '/views'
    ],
    'spiral'   => [
        directory("application") . '/views/spiral',
        directory("libraries") . '/spiral/framework/source/views'
    ],
    'profiler' => [
        directory("libraries") . '/spiral/profiler/source/views'
    ]
],
```

You can specify view name with a namespace using the ":" separator. If you don't enter a namespace, the "default" will be used instead. This makes it possible to rewrite the first example:

```php
public function index()
{
    return $this->views->render('default:demo', [
        'value' => mt_rand(0, 1000)
    ]);
}
```

The great thing about namespaces is that because ViewManager will be looking for file in the order of the declared directories, we can change the views used by other spiral components or modules. In this case, let's remember http configuration:

```php
'httpErrors'   => [
    400 => 'spiral:http/badRequest',
    403 => 'spiral:http/forbidden',
    404 => 'spiral:http/notFound',
    500 => 'spiral:http/serverError',
]
```

As you can see, soft http error system will render the view located in the "spiral" namespace, which is associated with two directories:

```php
'spiral'   => [
    directory("application") . '/views/spiral',
    directory("libraries") . '/spiral/framework/source/views'
],
```

Let's try to alter the HTTP 404 error page by creating a view in 'application/views/spiral/http' directory. We are going to overwrite the 'notFound' page. HttpDispacher will provide us a "request" variable we can use in our view:

```php
Page not found: <?= $request->getUri() ?>
```

> We can also simply point http to another view. :)

Now, if we get a 404 error, our view file will be rendered. If you already got a 404 error before it might be  pre-compiled already with the original view. We can reset it by either disabling cache, flushing the content of "application/runtime/cache/views" (default cache directory) or by executing the command "views:compile" (run with -v options to get more details) which will re-compile the available view files.

## View Engines
Spiral ViewManager would be useless if you didn't have the ability to create or mount your own templating engines and compilers. Before we do that, let's check out how spiral work swith view files and review configuration for the default spiral engine:

```php
'engines'      => [
    'default' => [
        'extensions' => [
            'php'
        ],
        'compiler'   => 'Spiral\Views\Compiler',
        'view'       => 'Spiral\Views\View'
    ]
],
```

The default engine can be associated with one of many filenames (we only use .php at the moment). Next we provide two classes used to describe every engine. Let's focus on these classes closely.

### View Compiler
The engine's Compiler option is responsible for the view source pre-compilation. This operation is done once and cached somewhere later. Spiral ViewManager leaves the cache management to Compiler class. Let's view CompilerInterface to see what methods are required from every compiler:

```php
interface CompilerInterface
{
    /**
     * @param ViewManager $views
     * @param FilesInterface $files
     * @param string $namespace View namespace.
     * @param string $view      View name.
     * @param string $filename  View filename.
     */
    public function __construct(
        ViewManager $views,
        FilesInterface $files,
        $namespace,
        $view,
        $filename
    );

    /**
     * @return string
     */
    public function getNamespace();

    /**
     * @return string
     */
    public function getView();

    /**
     * True if view has been compiled and cached somewhere already.
     *
     * @return bool
     */
    public function isCompiled();
    
    /**
     * @throws CompilerException
     * @throws \Exception
     */
    public function compile();

    /**
     * View filename location (to be rendered using require + export method or similar).
     *
     * @return string
     */
    public function compiledFilename();
}
```

Based on the given interface, we can see 3 primary methods ViewManager works with:
* isCompiler must return true if view source was processed and stored somewhere already
* compile method must process view source and preprate it for usage in renderer even if it's already cached
* compiledFilename must return location where compiled view are stored in it

Because most of the existing templating engines convert internal format into plain php files almost all of them can be mounted as compilers.

> Compiler must handle caching and cache validations by itself. However it is recommended that you read about view cache dependecies below. Compiler class doesn't have access to view variables.

### View Renderer
The second engine class declared under index 'view' must accept the instance of engine compiler and render `compiledFilename()`. When you want your view class to work with compilers, you have to implement `CompilerAwareInterface` which has a pre-defined set of arguments for it's constructor:

```php
interface CompilerAwareInterface extends ViewInterface
{
    /**
     * @param ContainerInterface $container
     * @param CompilerInterface  $compiler
     * @param array              $data
     */
    public function __construct(
        ContainerInterface $container,
        CompilerInterface $compiler,
        array $data = []
    );
}
```

In many cases, compiler will create a simple php file with placeholders for php variables. This makes it possible for us to use the existing `Spiral\Views\View` implementation. Let's looks into an implementation like this to understand how it works:

```php
class View extends Component implements CompilerAwareInterface
{
    /**
     * For render benchmarking.
     */
    use BenchmarkTrait;

    /**
     * @var array
     */
    protected $data = [];

    /**
     * @invisible
     * @var ContainerInterface
     */
    protected $container = null;

    /**
     * @invisible
     * @var CompilerInterface
     */
    protected $compiler = null;

    /**
     * {@inheritdoc}
     */
    public function __construct(
        ContainerInterface $container,
        CompilerInterface $compiler,
        array $data = []
    ) {
        $this->container = $container;
        $this->compiler = $compiler;
        $this->data = $data;
    }

    /**
     * Alter view parameters (should replace existing value).
     *
     * @param string $name
     * @param mixed  $value
     * @return $this
     */
    public function set($name, $value)
    {
        $this->data[$name] = $value;

        return $this;
    }

    /**
     * Set view rendering data. This will replace the full dataset.
     *
     * @param array $data
     * @return $this
     */
    public function setData(array $data)
    {
        $this->data = $data;

        return $this;
    }

    /**
     * {@inheritdoc}
     */
    public function render()
    {
        $__benchmark__ = $this->benchmark(
            'render',
            $this->compiler->getNamespace()
            . ViewsInterface::NS_SEPARATOR
            . $this->compiler->getView()
        );

        $__outputLevel__ = ob_get_level();
        ob_start();

        extract($this->data, EXTR_OVERWRITE);
        try {
            require $this->compiler->compiledFilename();
        } finally {
            while (ob_get_level() > $__outputLevel__ + 1) {
                ob_end_clean();
            }

            $this->benchmark($__benchmark__);
        }

        return ob_get_clean();
    }

    /**
     * {@inheritdoc}
     */
    final public function __toString()
    {
        return $this->render();
    }
}
```

As you can see, this utilizes classical php methods to render the php files using buffers and the `extract` function.

### Compilerless Renderers
There are scenarious where you might want to skip the compilation part and work with the view file directly. You can do that by implementing `FilenameAwareInterface` into your view class. In this case, additonal filename arguments will be provided to your class and point to view filename.

```php
interface FilenameAwareInterface extends ViewInterface
{
    /**
     * @param ContainerInterface $container
     * @param ViewManager        $views
     * @param string             $namespace
     * @param string             $view
     * @param string             $filename
     * @param array              $data
     */
    public function __construct(
        ContainerInterface $container,
        ViewManager $views,
        $namespace,
        $view,
        $filename,
        array $data = []
    );
}
```

> If you want, you can still handle caching and compilations inside your view.

## Spiral View Compiler
The default spiral compiler uses `Spiral\Views\View` for rendering purposes. Previously we discussed it's abilities so we will focus mainly on the compiler itself. The spiral compiler is represented by the `Spiral\Views\Compiler` class and built on the principles of the view processors. Essentially, it will pass view source through a set of classes dedicated to perform some code/source manipulations. Every processor must implement `ProcessorInterface`. Let's check out it's declaration:

```php
interface ProcessorInterface
{
    /**
     * @param ViewManager $views
     * @param Compiler    $compiler
     * @param array       $options
     */
    public function __construct(ViewManager $views, Compiler $compiler, array $options);

    /**
     * Compile view source.
     *
     * @param string $source View source (code).
     * @return string
     * @throws CompilerException
     * @throws \ErrorException
     */
    public function process($source);
}
```

If we wanted to add a custom processor into our compiler, all we need to do is modify the compiler section in our view configuration:

```php
'processors' => [
    'Spiral\Views\Processors\ExpressionsProcessor' => [],
    'Spiral\Views\Processors\TranslateProcessor'   => [],
    'Spiral\Views\Processors\TemplateProcessor'    => [],
    'Spiral\Views\Processors\EvaluateProcessor'    => [],
    'Spiral\Views\Processors\PrettifyProcessor'    => [],
    'Spiral\Toolkit\ResourceManager'               => []
]
```

> Every compiler class is associated with it's options. The default spiral processors lets you overwrite their options. We will skip this step for now.

Let's say that we want to create simple processor to replace `{{variable}}` with `<?=e($variable)?>`. We can name it `CoolEchoProcessor`:

```php
class CoolEchoProcessor implements ProcessorInterface
{
    /**
     * @param ViewManager $views
     * @param Compiler    $compiler
     * @param array       $options
     */
    public function __construct(ViewManager $views, Compiler $compiler, array $options)
    {
        //We don't have any options to be configured, so let's keep constructor empty
    }

    /**
     * Compile view source.
     *
     * @param string $source View source (code).
     * @return string
     * @throws CompilerException
     * @throws \ErrorException
     */
    public function process($source)
    {
        return preg_replace(
            '/{{([^}]+)}}/',
            '<?=e($\1)?>',
            $source
        );
    }
}
```

Now we can modify views config (to add new processor), our demo view file and see result in browser:

```php
This is demo view file: {{value}}

<?php
/**
 * @var \Spiral\Debug\Dumper $dumper
 */
$dumper = $this->container->get(\Spiral\Debug\Dumper::class);
$dumper->dump($value);
?>
```

If you want to see how compiled view file looks like, simply open 'application/runtime/cache/views/default-demo-*.php' file:

```php
This is demo view file: <?=e($value)?>
<?php
/**
 * @var \Spiral\Debug\Dumper $dumper
 */
$dumper = $this->container->get(\Spiral\Debug\Dumper::class);
$dumper->dump($value);
?>
```

Spiral Compiler ships with multiple pre-create processors used to simplify view files:

| Processor                                    | Description            
| ---                                          | ---
| Spiral\Views\Processors\ExpressionsProcessor | Runs few expressions defined in it's options, the most notable one @{variable} will replace it's value with cache dependency (read below).
| Spiral\Views\Processors\TranslateProcessor   | Replace `[[string]]` with their translations using active language, result are cached and depends on view cache dependencies (read below).
| Spiral\Views\Processors\TemplateProcessor    | Spiral templates allows user to inherit view layouts, create virual tags and much more. See [Templater] (/templater/overview.md).
| Spiral\Views\Processors\EvaluateProcessor    | Executes PHP blocks marked with #compile comment, used to perform some layout related code which does not depends on user input or view variables.
| Spiral\Views\Processors\PrettifyProcessor    | Removes emply view lines and spaces in some tag attributes.

## Cache and Cache Dependencies
Default spiral compiler uses cache configuration declared in view config, you can disable cache in development enviroment to get view updates immidiatelly, however it might slow down your application a lot (really a LOT):

```php
'cache'        => [
    'enabled'   => true,
    'directory' => directory("cache") . '/views'
],
```

As mention already compiler and it's processors does not have any access to view variables and user input, however `ViewManager` defines set of so called cache dependies declared in view configuration file:

```php
'dependencies' => [
    'language' => [
        'i18n',
        'getLanguage'
    ],
    'basePath' => [
        'http',
        'basePath'
    ]
],
```

Such depencies (in our case language - linked to current language, and basePath - linked to http base path values) will be used by default Compiler to change cached filename. Such technique provides us ability to use system functions (for example translations) on compilation stage without wasting CPU resources during rendering.

To demonstrate how cache dependecies work let's modify our demo view this way:

```php
[[This is demo view file:]] <?= $value ?>
```

As mention in processor section above every string embraced with `[[]]` will be automatically translated using active language, if we will open our controller action with view in browser we will see that no `[[]]` presented in output. Now, let's shitch our language and see what will happen:

```php
public function index()
{
    //Belarusian
    $this->i18n->setLanguage('by');

    return $this->views->render('default:demo', [
        'value' => mt_rand(0, 1000)
    ]);
}
```

As you might notice page rendering took few extra milliseonds on first run, this happen because Compiler created new cached view. To make example move visual let's edit our i18n string in 'application/runtime/i18n/belarus/view-demo.php' file (we can also export language into PO file using `i18n:export` command and then import edited file back using `i18n:import`).

```php
<?php return [
    'This is demo view file:' => 'This is demo view file specially for Belarus people:',
];
```

Now if we will reset or disable cache our output will demonstrate new content. To see what happen inside let's open directory 'application/runtime/cache/views'. As you can see our demo view now represented by 2 filename with different postfixes, if we will compare content of such files we will see the difference:

```php
This is demo view file: <?= $value ?>
```

```php
This is demo view file specially for Belarus people: <?= $value ?>
```

Such technique allows you to "capture" as many strings for localization as you want without paying for it with CPU resources.
