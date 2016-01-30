# Translator
Spiral Translate component utilizes Symfony\Translation interface and message formatters but simplifies logic on how locale and domains work.

## Translator Interface (expanded)

```php
interface TranslatorInterface //extends \Symfony\Component\Translation\TranslatorInterface
{
    /**
     * Default translation bundle.
     */
    const DEFAULT_DOMAIN = 'messages';

    /**
     * Default set of braces to be used in classes or views for indication of translatable content.
     */
    const I18N_PREFIX  = '[[';
    const I18N_POSTFIX = ']]';

    /**
     * Translates the given message.
     *
     * @param string      $id         The message id (may also be an object that can be cast to string)
     * @param array       $parameters An array of parameters for the message
     * @param string|null $domain     The domain for the message or null to use the default
     * @param string|null $locale     The locale or null to use the default
     *
     * @throws \InvalidArgumentException If the locale contains invalid characters
     *
     * @return string The translated string
     */
    public function trans($id, array $parameters = array(), $domain = null, $locale = null);

    /**
     * Translates the given choice message by choosing a translation according to a number.
     *
     * @param string      $id         The message id (may also be an object that can be cast to string)
     * @param int         $number     The number to use to find the indice of the message
     * @param array       $parameters An array of parameters for the message
     * @param string|null $domain     The domain for the message or null to use the default
     * @param string|null $locale     The locale or null to use the default
     *
     * @throws \InvalidArgumentException If the locale contains invalid characters
     *
     * @return string The translated string
     */
    public function transChoice($id, $number, array $parameters = array(), $domain = null, $locale = null);

    /**
     * Sets the current locale.
     *
     * @param string $locale The locale
     *
     * @throws \InvalidArgumentException If the locale contains invalid characters
     */
    public function setLocale($locale);

    /**
     * Returns the current locale.
     *
     * @return string The locale
     */
    public function getLocale();

    /**
     * Resolve domain name for given bundle.
     *
     * @param string $bundle
     * @return string
     */
    public function resolveDomain($bundle);

    /**
     * Get list of supported locales.
     *
     * @return array
     */
    public function getLocales();
}
```

The biggest implementation difference is located in how spiral process fallback locales and domains. First of all there is only one fallback locale per translaton, secondly spiral helps to route multiple "bundles" into one domain (see below).

## Usage
Classical usage of TranslatorInterface might looks like:

```php
public function indexAction(TranslatorInterface $translator)
{
    echo $translator->trans('Hello, {name}!', [
        'name' => $this->faker->name
    ]);
}
```

You can also use `transChoice` method of translator same way as it was designed in original interface: 

```php
public function indexAction(TranslatorInterface $translator)
{
    $count = mt_rand(0, 10);
    echo $translator->transChoice(
        '{0} There are no apples|{1} There is one apple|]1,Inf[ There are {count} apples',
        $count,
        compact('count')
    );
}
```

### Automatic message registration
If requested message not found in current and fallback locale, message key will be used by itself. In addition you set configuration flag "autoRegister" to true in your transaltor config, it will allow spiral to automatically add such message into fallback locale to be later exported into user friendly format (po, xml and etc).

## Short functions
To simplify developlement spiral provide two simple functions which are being aliases for trans and transChoice methods of active translator instance (resolved via static/shared container):

```php
public function indexAction()
{
    echo l('hello world1');
    
    $count = mt_rand(0, 10);
    echo p(
        '{0} There are no apples|{1} There is one apple|]1,Inf[ There are {count} apples',
        $count,
        compact('count')
    );
}
```

## Translator trait and domain routing
In many cases you might want to give ability to your class (component, controller, service) to translate messages
without reguesting trasnlator interface every time, such trait will work via instance related (or shared/static as fallaback) container:

```php

class HomeController extends Controller
{
    use TranslatorTrait;

    public function indexAction()
    {
        echo $this->say('Hello world');
    }
}
```

### Domain routing
As might notice we did not declare specific domain for our say trait, once called current class name will be used as our "bundle", in our case it will be "controllers-home-controller", using translator config we can state that every message inside this controller must be stored in a domain "controller-messages":

```php
'domains'          => [
    'controller-messages' => [
        'controllers-home-conrtroller',
        
        //We can also use star syntax
        'controllers-*'
    ],
    
    'spiral'     => [
        'spiral-*',
        'view-spiral-*',
        /*{{domain.spiral}}*/
    ],
    'profiler'   => [
        'view-profiler-*',
        /*{{domain.profiler}}*/
    ],
    'views'      => [
        'view-*',
        /*{{domain.views}}*/
    ],

    //Fallback domain
    'messages'   => ['*'],
],
```

## Translating errors
As mention in section dedicated to DataEntity you can also automatically translate error messages realted to entity/request validations, let's take a look as this example:

```php
class SampleRequest extends RequestFilter
{
    /**
     * @var array
     */
    protected $schema = [
        'name'    => 'data:name',
        'status'  => 'data:status'
    ];

    /**
     * @var array
     */
    protected $setters = [
        'name'    => 'trim',
        'status'  => 'trim'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'    => [
            ['notEmpty', 'error' => '[[Name is required]]'],
        ],
        'status'  => [
            'notEmpty',
            ['in_array', ['active', 'disabled'], 'error' => '[[Invalid status value]]']
        ]
    ];
}
```

As you can see we defined two custom error messages for name and status validations, once error will be raised
inner validation logic will automatically strip [[ and ]] wrappers and translate error using current locale.

```php
public function indexAction(SampleRequest $request)
{
    dump($request->getErrors());
}
```

> Please note, that only messages embraces with [[ and ]] are treated to be localized.

## Automatic indexation
Spiral Translator component provides additional functionality which works via `ClassLocatorInterface` and `InvocationLocatorInterface` and provides us ability to automatically index all possible messages using static analysis:

* Invocations of `l` and `p` methods
* Invocations of `$this->say()` method
* All messages defined in default properties of classes used TranslatorTrait and embraced with [[ and ]]

To index your translation methods simply run: `spiral i18n:index` (you can use -vv flag to get more details)

## Translating view files
Most of your location process will happen inside your view files, instead of forcing to use translator interface
or short function every time you want to mark your text to be translated simply embrace your text with [[ and ]]
characters:

```php
<extends:layouts.basic title="[[This is our title]]"/>

<block:content>
    [[Hello World!]]
</block:content>
```

> Attention, translation process of your view file will happen at moment of compilation and will use different cached
versions for different locales, meaning are you not limited in perfomance so you can translate as much text in your views
as you want. See [Views and Engines](/components/views.md)

You can also use such methodic in you twig files, view messages will be automatically registered at moment of compilation.

## Adding more locales
If you wish to add more locates to your application, simply create a folder inside your `app/resouces/locales' directory.

```php
public function indexAction()
{
    dump($this->translator->getLocales());
}
```

Translator will cache list of available locates for performace reasons, to flush this cache simply run: `spiral i18n:reload`.

Inside your locale directory you only need to place domain specific localization file in one of allowed formats defined in translator config:

```php
'loaders'          => [
    'php' => Translation\Loader\PhpFileLoader::class,
    'csv' => Translation\Loader\CsvFileLoader::class,
    'po'  => Translation\Loader\PoFileLoader::class,
    /*{{loaders}}*/
],
```

Let's try to translate message for our controller (routed to domain "controllers"):

```php
class HomeController extends Controller
{
    use TranslatorTrait;

    public function indexAction()
    {
        echo $this->say('Hello world!');
    }
}
```

Fist of all let's create locale directory for russian language `app/resources/locales/ru`, then let's put our translation domain into it, we are going to choose php format for simplicity (`app/resources/locales/ru/controllers.php`):

```php
<?php
return [
    'Hello world!' => 'Привет мир!'
];
```

Now, switching locale must also change message our controller is producing:

```php
public function indexAction()
{
    $this->translator->setLocale('ru');

    echo $this->say('Hello world!');
}
```

> You will have to reset translation cache using `i18n:reload` command first, if you don't want to do it every time, simply set option `autoReload` to true in your translation config. 

## Exporting locales
One of the most important parts of any tanslation process is actual translation, let's try to export our locale messages in a user friendly format using command `i18n:dump`:

```
> spiral i18n:dump ru russian
Dump successfully completed.
```

Now we can locale exported dumpers in 'russian' directory, you can always select other exporting format by adding command option, let's try to export into gettext po files:

```
> spiral i18n:dump ru russian -d po
Dump successfully completed.
```

```
msgid ""
msgstr ""
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"
"Language: ru\n"

msgid "Hello world!"
msgstr "Привет мир!"
```

> You can find list of available dumpers or add new in translator config.

## Changing locale
As stated in previous example changing locale is one function call `setLocale`, hovewer it might be resonable to automatically set different locales for different users and regions, we can utilize functionality of Http Middlewares for that, let's try to select appropariate locale based on user request headers:

```php
class LocaleDetector extends Service implements MiddlewareInterface
{
    /**
     * {@inheritdoc}
     */
    public function __invoke(Request $request, Response $response, callable $next)
    {
        $supported = $this->translator->getLocales();

        foreach ($this->fetchLocales($request) as $locale) {
            if (in_array($locale, $supported)) {
                $this->translator->setLocale($locale);
                break;
            }
        }

        return $next(
            $request->withAttribute('locale', $this->translator->getLocale()),
            $response
        );
    }

    /**
     * @param Request $request
     * @return array
     */
    public function fetchLocales(Request $request)
    {
        $header = $request->getHeaderLine('accept-language');
        foreach (explode(',', $header) as $value) {

            if (strpos($value, ';') !== false) {
                yield substr($value, 0, strpos($value, ';'));
            }

            yield $value;
        }
    }
}
```
