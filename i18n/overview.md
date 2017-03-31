# Translator
Spiral Translate component utilizes Symfony\Translation interface and message formatter but simplifies logic of how locale and domains work.

## Translator Interface

```php
interface TranslatorInterface extends \Symfony\Component\Translation\TranslatorInterface
{
    /**
     * Default set of braces to be used in classes or views for indication of translatable content.
     */
    const I18N_PREFIX  = '[[';
    const I18N_POSTFIX = ']]';

    /**
     * Resolve domain name for given bundle.
     *
     * @param string $bundle
     *
     * @return string
     */
    public function resolveDomain(string $bundle): string;

    /**
     * Get list of supported locales.
     *
     * @return array
     */
    public function getLocales(): array;
}
```

The biggest implementation difference is located in how spiral process fallback locales and domains.

First of all there is only one fallback locale per translation (by default 'en'), 
secondly spiral helps to route multiple "bundles" into one domain (see below).

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
## Short functions
The Translator ships with two IoC scope specific methods `l` and `p` which work as bridge to 
`translator->trans` and `translator->transChoice` accordingly:

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

## Add more locales
To add more locates to your application, simply create a folder inside your `app/resources/locales' directory.

> For example file "app/resources/locales/ru/views.ru.po" will represent "views" bundle for russian language.

```php
public function indexAction()
{
    dump($this->translator->getLocales());
}
```

Translator will cache list of available locates for performance reasons, to flush this cache run: 
`spiral i18n:reload`.

Inside your locale directory you only need to place domain specific localization file in one of allowed formats defined in translator config:

```php
'loaders'          => [
    'php' => Translation\Loader\PhpFileLoader::class,
    'csv' => Translation\Loader\CsvFileLoader::class,
    'po'  => Translation\Loader\PoFileLoader::class,
    /*{{loaders}}*/
],
```

## Export Locate
One of the most important parts of any translation process is actual translation, let's try to export our locale messages in a user friendly format using command `i18n:dump`:

```
> spiral i18n:dump ru russian
Dump successfully completed using Symfony\Component\Translation\Dumper\PhpFileDumper
Output directory: \var\www\sample.dev\russian
```

Use `-d` options to select alternative dumper:

```
> spiral i18n:dump ru russian -d po
Dump successfully completed using Symfony\Component\Translation\Dumper\PoFileDumper
Output directory: \var\www\sample.dev\russian
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

> Read how to index and capture all i18n messages of your application in [indexation](/i18n/indexation.md) section.

## Change locale
You are able to change your application locate at any moment by calling method `setLocate` of translator:

```php
public function indexAction()
{
    $this->translator->setLocale('ru');
}
```

Alternative way might include middleware approach:

```php
class LocaleDetector extends Service implements MiddlewareInterface
{
    /**
     * {@inheritdoc}
     */
    public function __invoke(Request $request, Response $response, callable $next)
    {
        try {
            return $next($request, $response);
        } catch (\DomainException $e) {
            throw new ServerErrorException($e->getMessage());
        }

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
     *
     * @return \Generator
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

## Troubleshooting
If you experiencing issues with translator seeing your locations files, try run `spiral configure` to
rebuild your project.
