# Translator
The Translator component simplify the way how you can work with your locates. You do not need to manually
register every used i18n string, but rather analyze your project. 

What can be indexed:
* Invocations of `l` and `p` methods
* Invocations of `$this->say()` method
* All messages defined in default properties of classes used TranslatorTrait and embraced with [[ and ]]
* Views

To index your translation methods run: `spiral i18n:index` (you can use -vv flag to get more details)

```
> spiral i18n:index -vv
Scanning translate function usages...
[Indexer] Indexing usages of 'l' function.
[Indexer] Indexing usages of 'p' function.
[Indexer] Indexing usages of 'say' method (TranslatorTrait).
[Indexer] Found invocation of 'say' in '/var/www/spiral.dev/app/classes/Controllers/IndexController.php' at line 26.
[Indexer] Found [messages]: 'Hello World'
Scanning Translatable classes...
[Indexer] Found translation string(s) in class 'Requests\UploadRequest'.
[Indexer] Found [requests]: 'Please select as least one image'
[Indexer] Found translation string(s) in class 'Spiral\Validation\Checkers\AddressChecker'.
[Indexer] Found [validation]: 'Must be a valid email address.'
[Indexer] Found [validation]: 'Must be a valid URL address.'
[Indexer] Found translation string(s) in class 'Spiral\Validation\Checkers\FileChecker'.
[Indexer] Found [validation]: 'File does not exists.'
[Indexer] Found [validation]: 'File not received, please try again.'
[Indexer] Found [validation]: 'File exceeds the maximum file size of {1}KB.'
[Indexer] Found [validation]: 'File has an invalid file format.'
[Indexer] Found translation string(s) in class 'Spiral\Validation\Checkers\ImageChecker'.
[Indexer] Found [validation]: 'Image does not supported.'
[Indexer] Found [validation]: 'Image does not supported (allowed JPEG, PNG or GIF).'
[Indexer] Found [validation]: 'Image size should not exceed {0}x{1}px.'
[Indexer] Found [validation]: 'The image dimensions should be at least {0}x{1}px.'
[Indexer] Found translation string(s) in class 'Spiral\Validation\Checkers\MixedChecker'.
[Indexer] Found [validation]: 'Please enter valid card number.'
[Indexer] Found translation string(s) in class 'Spiral\Validation\Checkers\NumberChecker'.
[Indexer] Found [validation]: 'Your value should be in range of {0}-{1}.'
[Indexer] Found [validation]: 'Your value should be higher than {0}.'
[Indexer] Found [validation]: 'Your value should be lower than {0}.'
[Indexer] Found translation string(s) in class 'Spiral\Validation\Checkers\RequiredChecker'.
[Indexer] Found [validation]: 'This value is required.'
[Indexer] Found [validation]: 'This value is required.'
[Indexer] Found [validation]: 'This value is required.'
[Indexer] Found [validation]: 'This value is required.'
[Indexer] Found [validation]: 'This value is required.'
[Indexer] Found translation string(s) in class 'Spiral\Validation\Checkers\StringChecker'.
[Indexer] Found [validation]: 'This value is required.'
[Indexer] Found [validation]: 'Your value does not match required pattern.'
[Indexer] Found [validation]: 'Enter text shorter or equal to {0}.'
[Indexer] Found [validation]: 'Your text must be longer or equal to {0}.'
[Indexer] Found [validation]: 'Your text length must be exactly equal to {0}.'
[Indexer] Found [validation]: 'Text length should be in range of {0}-{1}.'
[Indexer] Found translation string(s) in class 'Spiral\Validation\Checkers\TypeChecker'.
[Indexer] Found [validation]: 'This value is required.'
[Indexer] Found [validation]: 'This value is required.'
[Indexer] Found [validation]: 'Not a valid boolean.'
[Indexer] Found [validation]: 'Not a valid datetime.'
[Indexer] Found [validation]: 'Not a valid timezone.'
[Indexer] Found translation string(s) in class 'Spiral\Validation\Validator'.
[Indexer] Found [validation]: 'Condition '{condition}' does not meet.'
```

> All found string will be places into proper domain based on your config settings.

By default all found strings will be stored in default locale, to export them into different language
use:

```
spiral i18n:dump ru russian -d po -f
Dump successfully completed using Symfony\Component\Translation\Dumper\PoFileDumper
Output directory: D:\WebServer\domains\spiral.dev\russian
```

> You can index your views by simply compiling them.