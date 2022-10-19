# Scaffolder
Default Skeleton application includes pre-installed scaffolder module with set of CLI commands used to generate parts of your application:

> You can locate module sources [here](https://github.com/spiral-modules/scaffolder).

## Installation
If you do not have scaffolder module installed execute following commands:

```
composer require spiral/scaffolder
spiral register spiral/scaffolder
```

## Available Commands
Command            | Description
---                | ---
create:command     | Create command declaration
create:controller  | Create controller declaration
create:document    | Create Document declaration
create:middleware  | Create middleware declaration
create:migration   | Create migration declaration
create:record      | Create Record declaration
create:request     | Create RequestFilter declaration
create:service     | Create service/model declaration

## Configuration
You can customize scaffolder component by replacing declaration generators and their options using `modules/scaffolder` configuration file:

```php
return [
    /*
     * This is set of comment lines to be applied to every scaffolded file, you can use env() function
     * to make it developer specific or set one universal pattern per project.
     */
    'header'       => [
        '{project-name}',
        '',
        '@author {author-name}'
    ],

    /*
     * Base directory for generated classes, class will be automatically localed into sub directory
     * using given namespace.
     */
    'directory'    => directory('application') . 'classes/',

    /*
     * Default namespace to be applied for every generated class.
     *
     * Example: 'namespace' => 'MyApplication'
     * Controllers: MyApplication\Controllers\SampleController
     */
    'namespace'    => '',

    /*
     * This is set of default settings to be used for your scaffolding commands.
     */
    'declarations' => [
        'controller'      => [
            'namespace' => 'Controllers',
            'postfix'   => 'Controller',
            'class'     => Declarations\ControllerDeclaration::class
        ],
        'service'         => [
            'namespace' => 'Models',
            'postfix'   => 'Service',
            'class'     => Declarations\ServiceDeclaration::class
        ],
        'middleware'      => [
            'namespace' => 'Middlewares',
            'postfix'   => '',
            'class'     => Declarations\MiddlewareDeclaration::class
        ],
        'command'         => [
            'namespace' => 'Commands',
            'postfix'   => 'Command',
            'class'     => Declarations\CommandDeclaration::class
        ],
        'request'         => [
            'namespace' => 'Requests',
            'postfix'   => 'Request',
            'class'     => Declarations\RequestDeclaration::class,
            'options'   => [
                //Set of default filters and validate rules for various types
                'mapping' => [
                    'int'    => [
                        'source'    => 'data',
                        'setter'    => 'intval',
                        'validates' => ['notEmpty', 'integer']
                    ],
                    'float'  => [
                        'source'    => 'data',
                        'setter'    => 'floatval',
                        'validates' => ['notEmpty', 'float']
                    ],
                    'string' => [
                        'source'    => 'data',
                        'setter'    => 'strval',
                        'validates' => ['notEmpty', 'string']
                    ],
                    'bool'   => [
                        'source'    => 'data',
                        'setter'    => 'boolval',
                        'validates' => ['notEmpty', 'boolean']
                    ],
                    'email'  => [
                        'source'    => 'data',
                        'setter'    => 'strval',
                        'validates' => ['notEmpty', 'string', 'email']
                    ],
                    'file'   => [
                        'source'    => 'file',
                        'validates' => ['file::uploaded']
                    ],
                    'image'  => [
                        'source'    => 'file',
                        'validates' => ["image::uploaded", "image::valid"]
                    ],
                    /*{{request.mapping}}*/
                ]
            ]
        ],
        'record'          => [
            'namespace' => 'Database',
            'postfix'   => '',
            'class'     => Declarations\Database\RecordDeclaration::class
        ],
        'document'        => [
            'namespace' => 'Database',
            'postfix'   => '',
            'class'     => Declarations\Database\DocumentDeclaration::class
        ],
        /*{{elements}}*/
    ],
];
```