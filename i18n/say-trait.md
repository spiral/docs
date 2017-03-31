
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