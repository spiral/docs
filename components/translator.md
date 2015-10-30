# Translator
Spiral Translator component provides ability to create, fetch and manage your application translation text. Generic principle of translator is to organize set of strings to be translated into one common storage file called "string bundle". In addtion to that, spiral framework provides set of tools which can simplify development process and exclude step of registering string in bundle files manually.

## TranslatorInterface
You can always retrieve instance of Translator component using either `TranslatorInterface` dependency or short binding "i18n", let's check `TranslatorInterface` source to see what methods are available for us:

```php
interface TranslatorInterface
{
    /**
     * Default translation bundle.
     */
    const DEFAULT_BUNDLE = 'default';

    /**
     * Default set of braces to be used in classes or views to indicate translatable content.
     */
    const I18N_PREFIX  = '[[';
    const I18N_POSTFIX = ']]';

    /**
     * Change language.
     *
     * @param string $language
     * @throws LanguageException
     */
    public function setLanguage($language);

    /**
     * Get current language.
     *
     * @return string
     */
    public function getLanguage();

    /**
     * Translate value using active language. Method must support message interpolation using
     * interpolate method or sptrinf.
     *
     * Examples:
     * $translator->translate('bundle', 'Some Message');
     * $translator->translate('bundle', 'Hello %s', $name);
     *
     * @param string      $bundle
     * @param string      $string
     * @param array|mixed $options Interpolation options.
     * @return string
     * @throws TranslatorException
     */
    public function translate($bundle, $string, $options = []);

    /**
     * Pluralize string using language pluralization options and specified numeric value. Number
     * has to be ingested at place of {n} placeholder.
     *
     * Examples:
     * $translator->pluralize("{n} user", $users);
     *
     * @param string $phrase Should include {n} as placeholder.
     * @param int    $number
     * @param bool   $format Format number.
     * @return string
     * @throws TranslatorException
     */
    public function pluralize($phrase, $number, $format = true);
}
```

As you can see, interface declares only few methods without creating overcomplicated flow, let's try to demostrate few translator abilities using controller action:

```php
protected function indexAction()
{
    //Current language code
    dump($this->i18n->getLanguage());

    //We are going to use current class name as bundle name
    dump($this->i18n->translate(
        self::class, 'Welcome to HomeController, {name}.', ['name' => 'User']
    ));

    dump($this->i18n->pluralize('{n} user', mt_rand(0, 1000)));
}
```

First of all we can show current language ID, in our case it will be "en". Next we can try to translate and interpolate string 'Welcome to HomeController, {name}.' using bundle named as our parent class, this means that our translation information are isolated from other application parts and we need to load it only in specific cases. At the end we are trying to build pluraized phase using given number and template.

One this code will be executed we can open our `application/runtime/i18n/en/` directory and locate 2 generated bundle files:

Controllers-HomeController.php

```php
<?php
return [
    'Welcome to HomeController, {name}.' => 'Welcome to HomeController, {name}.',
];
```

plural-phrases.php

```php
<?php
return [
    '{n} user' => [
        0 => '{n} user',
        1 => '{n} user',
    ],
];
```

Now we can edit plural-phases file to ensure that plural phase has valid form:

```php
<?php
return [
    '{n} user' => [
        0 => '{n} user',
        1 => '{n} users',
    ],
];
```

> All application plural phrases are aggreaged in one bundle. Switching to different languages will create plural bundles with different rules and phrase counts (for example russian has 3 word forms for plurals).

## Using Short functions
In many cases you might wish to translate or pluralize string without specifying bundle, for this purposes you can use 2 short function `l` and `p`. Now we can rewrite our controller code to use such functions:

```php
protected function indexAction()
{
    //Current language code
    dump($this->i18n->getLanguage());

    //We are going to use current class name as bundle name
    dump(l('Welcome to HomeController, {name}.', ['name' => 'User']));

    dump(p('{n} user', mt_rand(0, 1000)));
}
```

As you can see, now we don't need to specify bundle name for `l` function as `TranslatorInterface::DEFAULT_BUNDLE` (default) will be used by default. Pluralizarion function `p` will behave exacly same way as before. 

> Attention, both functions will access Translator instance using global container, if you don't have global container enabled (always on in spiral framework application) your code will fail.

## TranslatorTrait
Spiral framework provide much more simplistic way to localize string inside given classes, to access such functionality we can simply need use trait `TranslatorTrait` in our class (attention, trait will talk to Translator instance using it's singleton and global contrainer).

```php
class HomeController extends Controller
{
    use TranslatorTrait;

    protected function indexAction()
    {
        //We are going to use current class name as bundle name
        dump(self::translate('Welcome to HomeController, {name}.', ['name' => 'User']));
    }
}
```

Given trait will declare static method "translate" available for you and similar by usage to short function `l`, however class name will be used instead of default bundle.

TranslatorTrait already used by DataEntities (ORM and ODM) and Validator checkers to generate language specific set of error messages.

> Method is static only to allow system index it's usages and generate bundle files without manual intervention.

## Indexing translation
One of the important parts of Translator component is ability to index strings without executing target code or manually editing bundle files. This funationality has only one limitation - only static methods or short functions can be indexed and only in case when their string argument are constant. Let's view example of what specific parts of our code can be indexed:

```php
protected function indexAction()
{
    //Can be indexed
    self::translate('Welcome to HomeController, {name}.', ['name' => 'Anton']);
    l('Some string');
    p('{n} user', mt_rand(0, 100));
    
    $string = 'Some string';
    
    //Ca not be indexed
    $this->translate('Some string');
    self::translate($string);
}
```

To run indexation we can execute console command `i18n:index` (execute command with higher verbosity to see more details). In cases where translate method will be feeded with different non well determinated strings (for example error messages translation), we can use analyzation hack which provides us ability to index messages declated in default class properties. To do that, we can have to put string messsage somewhere in class property and embrace it with `[[` and `]]`, let's view an example:

```php
class HomeController extends Controller
{
    use TranslatorTrait;

    protected $strings = [
        'good' => '[[Everything is good.]]',
        'bad'  => '[[Bad!]]'
    ];

    protected function indexAction()
    {
        $this->translate($this->strings['good']);
        $this->translate($this->strings['bad']);
    }
}
```

Both good and bad message can be indexed using console command without executing `HomeController`.

> Such technique are mainly used to specify model validation error messages.

## Localizing Views
Spiral [ViewManager] (/components/views.md) has pre-built view source processor dedicated to automatically translate view strings, string much follow same format as in default class properties and be embraced with `[[` and `]]` symbols.

```html
<div>[[Welcome to our view!]]</div>
```

One notable part of view localization is that view manager will use current translator language as **cache dependency**, meaning string translation are performed at compilation and not runtime phase of view rendering, this way provides you ability to translate and "capture" as many strings as you want without any performance drop.

## Same Language Translation
One notable aspect of covering as much view text with translator tags as possible is that such text can be edited from client side (for example by using admin panel, coming soon) to translate weird programmers language into normal one.
