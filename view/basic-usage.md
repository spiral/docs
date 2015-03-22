# Basic Usage of View component
By default view component provides very basic view rendering abilities, such abilities includes executing
of php templates with provided set of variables. To initate view rendering use following code (example given for controller
action):
```php
public function action()
{
    return $this->view->render('view');
}
```
You can also use View facade:
```php
use \Spiral\Facades\View;

View::render('view');
```
To pass view variables use second argument of render method.
```php
public function action()
{
    return $this->view->render('view', [
        'name' => 'value'
    ]);
}
```
Based on provided code, framework will try to find view file located in default view directory (specified in configuration of view component) `application/views/view.php`. Such view can be simple php file with inline HTML:
```php
<html>
<body>
    This is <?= $value ?>
</body>
</html>
```
As you can see provided array of view data was exported to be used as local view variables.
> Spiral does not use any "logic" templater like Smarty or Twig, instead it uses HTML composer (see Templater section).
If you want to connect custom templating engine, you may want to register custom view Processor.

## Namespaces
All view files are separated by set of namespaces listed in view configuration (`application/config/view.php`) such namespaces
helps to separate view files between modules, core and application. Let's take a look on sample view config:
```php
'namespaces'        => array(
    'default'  => array(
        directory("application") . '/views/'
    ),
    'spiral'   => array(
        directory("application") . '/views/spiral/',
        directory("framework") . '/views/'
    )
)
```
To render view file from dedicated namespace, we can use syntax where namespaces separated by `:` with view name:
```php
$this->view->render('spiral:http/serverError');
```
> Spiral will always force `default` namespace if nothing else provided.

Provided code will try to locate view named `serverError` in directory `application/views/spiral/` and then in directory `framework/views`. Such organization gives us ability to redefine view files declared in other namespaces by declaring our namespace directory with higher priority.

> Besides `http/serverError` you can redefine `http/notFound`, `http/forbidden`, `http/badRequest` views.

## Redering Process
To better understand how to extend view component let's review rendering process:

#### View filename lookup
First step of any view rendering process is to locate view file should be used to perform rendering. Location depends on provided namespace and set of namespace directories. 

#### Generating cached view filename
Once view file located, spiral will generate cache filename which is used or *will be used* to store cached view template. Generated name depends on environment and set of static variables provided to view component (see section static variables). 

#### View cache check
If view cache exsist spiral will make sure that original file wasn't altered since cache generation.
> You can disable cache and always invalidate view cache, this is recommended bahaviour during development as framework can't track updates in nested files. However it will dramatically slow your application. Never turn cache off in production.

#### View Compilation
Before final view rendering which will happen in runtime view component will pass source of found template thought set of defined view processors. Such processors will perform operations such as view localization, pretty print, html composition and etc. This is the slowest part view rendering process and can take up to 1 second for huge templates. Fortunatelly result of
compilatation are plan php file which is cached in dedicated filename.
> For big projects is reasonable to compile view content while deployment using `view:cache` command.

#### Rendering
Once cache filename located or created, runtime variables will be injected in it and file will be executed.

## Static Variables
As mentioned earlier view cache depends on set of static variables, such variables designed to be used only while compilation stage and **should not** include any user data. Consider them as compilation options. Different set of variables will be generating different view cache. To set variable use following code:
```php
$this->view->staticVariable('layout', 'mobile');
$this->view->staticVariable('authorized', true);

//Getting current variable value
dump($this->view->staticVariable('layout'));

$this->view->render('home');
```
To get access to variable in your view:
```php
This is @{layout|default-value}
```

## View Processors
Spiral supplied with set of view processors used to simplify view development, processors:

Processor                                               | Description
---                                                     | ---
Spiral\Components\View\Processors\VariablesProcessor    | Inject static view variables using `@{variable|default}` syntax.
Spiral\Components\View\Processors\LocalizationProcessor | Translate `[[string]]` patterns, result are cached.
Spiral\Components\View\Processors\TemplateProcessor     | Extend and composite view files, see Templater section to get more information.
Spiral\Components\View\Processors\EvaluateProcessor     | Execute PHP blocks marked with compilation flag and cache it's result.
Spiral\Components\View\Processors\ShortTagsProcessor    | Replace php tags `<?` with `<?php`. 
Spiral\Components\View\Processors\PrettyPrintProcessor  | Remove blank lines.
> You can create your own processor by implementing `Spiral\Components\View\ProcessorInterface` interface and registering it in view config.

## Localization
One of view processors used to localize view content based on current language setting provided by `Translator`.
To use localization processor simply update desired string with `[[` and `]]` symbols. As result of every processor compilation cached, there is no limitation of how many strings you can "capture" and no performance drop.
```php
<html>
<body>
    <p>[[This is]] <?= $value ?>.</p>
    <?='[[We can also use it in PHP.]]'?>
    <a href="">[[link]]</a>    
</body>
</html>
```
All captured string will be automatically added to i18n bundles at time of compilation.
> Translator will set static variable `language` which means that switching language will generate different view cache.

## Evaluator
Sometimes you may need to execute your PHP blocks only once, for example based on static variable value, to do that simply add any of this comments (`/*compile*/`, `#compile`, `#php-compile`) to part of your php block:
```php
<html>
<body>
    <p>[[This is]] <?= $value ?>.</p>
    <?='[[We can also use it in PHP.]]'?>
    <a href="/" title="[[Title string.]]">[[link]]</a>    
    <br/>
    <?php #compile
        echo date("Y m, H:i");
    ?>
</body>
</html>
```
Now, if we will check cached view file we will see something like that:
```php
<html>
<body>
    <p>This is <?= $value ?>.</p>
    <?='We can also use it in PHP.'?>
    <a href="/" title="Title string.">link</a>    
    <br/>
    2015 03, 20:10
</body>
</html>
```
As you can see all localized strings are replaced with their transaltions and php block executed, however all other php blocks still presented in untouched form.
