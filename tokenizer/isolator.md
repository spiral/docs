# PHP Blocks Isolation
In some cases you might want to remove or temporary hide PHP blocks located in a given string/file. Use `Spiral\Tokenizer\Isolator` class for such purposes. 

> Note, this class works thought php tokens.

## Examples
Remove all php blocks in string:

```php
public function indexAction()
{
    $code = 'hello <?=$name?>';

    $isolator = new Isolator();
    $code = $isolator->isolatePHP($code);

    dump($code);
}
```

Such approach useful when you want to pre-process string or view source without corrupting inner php code, to restore code which been previously isolated call method `repairPHP`:

```php
public function indexAction()
{
    $code = 'hello <?=$name?>';

    $isolator = new Isolator();
    $code = $isolator->isolatePHP($code);

    //Do some manipulations with code
    
    $code = $isolator->repairPHP($code);
    
    dump($code);
}
```

You are also able to get list of all php block found in a given string and replace their values:

```php
public function indexAction()
{
    $code = 'hello <?=$name?>';

    $isolator = new Isolator();
    $code = $isolator->isolatePHP($code);

    foreach ($isolator->getBlocks() as $id => $block) {
        $isolator->setBlock($id, '<?=e($name)?>');
    }

    $code = $isolator->repairPHP($code);

    dump($code);
}
```