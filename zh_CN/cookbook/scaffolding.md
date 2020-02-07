# 速查手册 - 脚手架

## 安装

可以通过 composer 安装此扩展：

```bash
$ composer require spiral/scaffolder
```

然后确认在你的 App 类中加入 `Spiral\Scaffolder\Boot\Bootloader\ScaffolderBootloader`:

```php
use Spiral\Scaffolder\Bootloader\ScaffolderBootloader;

class App extends Kernel
{
    // ...

    protected const APP = [
        // ...
        ScaffolderBootloader::class
    ];
}
```

> 注意，此扩展会调用 `TokenizerConfig`, 所以请把它添加到 bootload 链的最后。

## 配置

You can customize scaffolder component by replacing declaration generators and their options using `scaffolder` configuration file.
Default configuration is located in the `ScaffolderBootloader`

您可以使用 `scaffolder` 配置文件（`app/config/scaffolder.php`）替换声明生成器及其选项，从而自定义 scaffolder 组件。
默认配置位于可以在 `ScaffolderBootloader` 中找到。


## 可用命令

命令            | 说明
---                | ---
create:bootloader  | 创建引导程序
create:command     | 创建命令
create:config      | 创建配置类
create:controller  | 创建控制器
create:filter      | 创建 HTTP 请求过滤器
create:jobHandler  | 创建任务处理器
create:middleware  | 创建中间件
create:migration   | 创建数据库迁移
create:repository  | 创建实体仓库
create:entity      | 创建实体

### 创建引导程序类

```bash
$ php app.php create:bootloader <name>
```

此命令会创建名为 `<Name>Bootloader` 的类。

#### 示例

```bash
$ php app.php create:bootloader my
```

创建的类（`<app directory>\src\Bootloader\MyBootloader.php`）代码如下：

```php
use Spiral\Boot\Bootloader\Bootloader;

class MyBootloader extends Bootloader
{
    protected const BINDINGS = [];

    protected const SINGLETONS = [];

    protected const DEPENDENCIES = [];

    public function boot(): void
    {
    }
}
```

### 创建命令

```bash
$ php app.php create:command <name> [alias]
```

此命令会创建 `<Name>Command` 类。命令的执行名称是 `name` 或者 `alias`(如果指定).

#### 不指定 `alias` 的示例

```bash
$ php app.php create:bootloader my
```

生成的类（`<app directory>\src\Command\MyCommand.php`）代码如下
```php
use Spiral\Console\Command;

class MyCommand extends Command
{
    protected const NAME = 'my';

    protected const DESCRIPTION = '';

    protected const ARGUMENTS = [];

    protected const OPTIONS = [];

    /**
     * Perform command
     */
    protected function perform(): void
    {
    }
}
```

#### 指定 `alias` 的示例：
```bash
$ php app.php create:bootloader my alias
```

生成的类（`<app directory>\src\Command\MyCommand.php`）代码如下
```php
use Spiral\Console\Command;

class MyCommand extends Command
{
    protected const NAME = 'alias';

    protected const DESCRIPTION = '';

    protected const ARGUMENTS = [];

    protected const OPTIONS = [];

    /**
     * Perform command
     */
    protected function perform(): void
    {
    }
}
```

### 创建配置类
```bash
$ php app.php create:config <name>
```

此命令会创建名为 `<Name>Config` 的类。同时，如果对应的配置文件 `<app directory>/config/<name>.php` 不存在的话，也会自动创建。

可用选项:
* `--reverse (-r)` - 指定此标志时, 脚手架会查找 `<app directory>/config/<name>.php` 文件并基于该文件创建对应的 `<Name>Config` 类。
生成的类会包含默认值和 getters, 有时它还会为数组生成 by-key-getters. 具体如下：<br/>

如果数值值包含有多个子值，且所有子值的具有相同类型的键和值时，脚手架会尝试创建 by-key-getter 方法。
如果要生成的方法名称已经存在，by-key-getter 不会创建。

#### 空配置文件示例：
```bash
$ php app.php create:config my
```

生成的配置文件（`<app directory>/config/MyConfig.php`）代码如下：
```php
return [];
```

生成的类（`<app directory>/src/Config/MyConfig.php`）代码如下：
```php
use Spiral\Core\InjectableConfig;

class MyConfig extends InjectableConfig
{
    public const CONFIG = 'my';

    /**
     * @internal For internal usage. Will be hydrated in the constructor.
     */
    protected $config = [];
}
```

#### 反向生成示例
```php
//...存在的 "my.php" 配置文件：
return [
    //将会创建 "getParam()" by-key-getter 方法（单数名称转换成功）
    'params'      => [
        'one' => 'param',
        'two' => 'another param',
    ],
    //将会创建 "getParameterBy()" by-key-getter 方法（单数名称转换不成功）
    'parameter'   => [
        'one' => 'parameter',
        'two' => 'another parameter',
    ],
    //将会创建 "getValueBy()" by-key-getter 方法（因为 "getValue()" 会与下面的 "value" 字段冲突)
    'values'      => [
        1 => 'value',
        2 => 'another value',
    ],
    'value'       => 'third value',
    //不会创建 by-key-getter 方法，因为只有一个子值
    'few'        => [
        'one' => 'value',
    ],
    //不会创建 by-key-getter 方法，因为有不同类型的子值
    'mixedValues' => [
        'one' => 'value',
        'two' => 2,
    ],
    //不会创建 by-key-getter 方法，因为有不同类型的子健
    'mixedKeys'   => [
        'one' => 'value',
        2     => 'another value',
    ],
    //不会创建 by-key-getter 方法
    //（因为 "getConflict()" 和 "getConflictBy()" 分别已经被接下来的 "conflict" 和 "conflictBy" 字段占用）
    'conflicts'   => [
        'one' => 'conflict',
        'two' => 'another conflict',
    ],
    'conflict'    => 'third conflic',
    'conflictBy'  => 'fourth conflic',
];
```

```bash
$ php app.php create:config my -r
```

生成的类（`<app directory>/src/Config/MyConfig.php`）代码如下：
```php
use Spiral\Core\InjectableConfig;

class MyConfig extends InjectableConfig
{
    public const CONFIG = 'my';

    /**
     * @internal For internal usage. Will be hydrated in the constructor.
     */
    protected $config = [
        'params'      => [],
        'parameter'   => [],
        'values'      => [],
        'value'       => '',
        'few'         => [],
        'mixedValues' => [],
        'mixedKeys'   => [],
        'conflicts'   => [],
        'conflict'    => '',
        'conflictBy'  => ''
    ];

    /**
     * @return array|string[]
     */
    public function getParams(): array
    {
        return $this->config['params'];
    }

    /**
     * @return array|string[]
     */
    public function getParameter(): array
    {
        return $this->config['parameter'];
    }

    /**
     * @return array|string[]
     */
    public function getValues(): array
    {
        return $this->config['values'];
    }

    /**
     * @return string
     */
    public function getValue(): string
    {
        return $this->config['value'];
    }

    /**
     * @return array|string[]
     */
    public function getFew(): array
    {
        return $this->config['few'];
    }

    /**
     * @return array
     */
    public function getMixedValues(): array
    {
        return $this->config['mixedValues'];
    }

    /**
     * @return array|string[]
     */
    public function getMixedKeys(): array
    {
        return $this->config['mixedKeys'];
    }

    /**
     * @return array|string[]
     */
    public function getConflicts(): array
    {
        return $this->config['conflicts'];
    }

    /**
     * @return string
     */
    public function getConflict(): string
    {
        return $this->config['conflict'];
    }

    /**
     * @return string
     */
    public function getConflictBy(): string
    {
        return $this->config['conflictBy'];
    }

    /**
     * @param string param
     * @return string
     */
    public function getParam(string $param): string
    {
        return $this->config['params'][$param];
    }

    /**
     * @param string parameter
     * @return string
     */
    public function getParameterBy(string $parameter): string
    {
        return $this->config['parameter'][$parameter];
    }

    /**
     * @param int value
     * @return string
     */
    public function getValueBy(int $value): string
    {
        return $this->config['values'][$value];
    }
}
```

### 创建控制器
```bash
$ php app.php create:controller <name>
```

此命令会创建名为 `<Name>Controller` 的类。 可用选项：
* `--action (-a)` (允许多次使用) - 用此选项添加 action 方法
* `--prototype (-p)` - 如果指定此标志，会添加 `PrototypeTrait`

#### 不添加 action 的示例：
```bash
$ php app.php create:controller my
```

生成的类（`<app directory>/src/Controller/MyController.php`）代码如下：
```php
class MyController
{
}
```

#### 使用 `--prototype` 标志的示例：
```bash
$ php app.php create:controller my -p
```

生成的类（`<app directory>/src/Controller/MyController.php`）代码如下：
```php
use Spiral\Prototype\Traits\PrototypeTrait;

class MyController
{
    use PrototypeTrait;
}
```

#### 包含 action 方法的示例：
```bash
$ php app.php create:controller my -a index -a create -a update -a delete
```

生成的类（`<app directory>/src/Controller/MyController.php`）代码如下：
```php
class MyController
{
    public function index()
    {
    }

    public function create()
    {
    }

    public function update()
    {
    }

    public function delete()
    {
    }
}
```

### 创建 HTTP 请求过滤器
```bash
$ php app.php create:filter <name>
```

 此命令会创建名为 `<Name>Filter` 的类。可用选项：

* `--entity (-e)` - 你可以指定实体类 `EntityClass`, 过滤器生成器会取出指定实体类的所有属性，放入生成的过滤器
并尝试根据每个属性的类型声明(php7.4)、默认值或者 PhpDoc 定义它的类型。或者你也可以用 `field` 选项指定过滤器的模式（Schema）。
* `--field (-f)` -（可以多次使用）。完整的字段格式是 `name:type(source:origin)`.
`type`, `origin` 和 `source:origin` 是可选的，可以省略，默认值如下：
  * type=string
  * source=data
  * origin=\<name\>

> 了解更多有关过滤器的信息，可以查看 [filters](https://github.com/spiral/filters) 组件包

#### 不含字段定义的示例：
```bash
$ php app.php create:filter my
```

生成的类（`<app directory>/src/Filter/MyFilter.php`）代码如下：
```php
use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [];

    protected const VALIDATES = [];

    protected const SETTERS = [];
}
```

#### 包含字段定义的示例：
```bash
$ php app.php create:filter my -f unknown_val -f str_val:string -f int_val:int -f bool_val:bool(query:from_bool) -f float_val:float(query)
```

生成的类（`<app directory>/src/Filter/MyFilter.php`）代码如下：
```php
use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'unknown_val' => 'data:unknown_val',
        'str_val'     => 'data:str_val',
        'int_val'     => 'data:int_val',
        'bool_val'    => 'query:from_bool',
        'float_val'   => 'query:float_val'
    ];

    protected const VALIDATES = [
        'unknown_val' => [
            'notEmpty',
            'string'
        ],
        'str_val'     => [
            'notEmpty',
            'string'
        ],
        'int_val'     => [
            'notEmpty',
            'integer'
        ],
        'bool_val'    => [
            'notEmpty',
            'boolean'
        ],
        'float_val'   => [
            'notEmpty',
            'float'
        ]
    ];

    protected const SETTERS = [];
}
```

#### 指定实体源的示例：
```php
//...已经存在的 "MyEntity" 实体类：
class MyEntity
{
    protected bool $typedBool;

    public $noTypeString;

    /** @var SomeOtherEntity */
    public $obj;

    /** @var int */
    protected $intFromPhpDoc;

    private $noTypeWithFloatDefault = 1.1;
}
```

```bash
$ php app.php create:filter my -e MyEntity
```

生成的类（`<app directory>/src/Filter/MyFilter.php`）代码如下：
```php
use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'typedBool'              => 'data:typedBool',
        'noTypeString'           => 'data:noTypeString',
        'obj'                    => 'data:obj',
        'intFromPhpDoc'          => 'data:intFromPhpDoc',
        'noTypeWithFloatDefault' => 'data:noTypeWithFloatDefault'
    ];

    protected const VALIDATES = [
        'typedBool'              => [
            'notEmpty',
            'boolean'
        ],
        'noTypeString'           => [
            'notEmpty',
            'string'
        ],
        'obj'                    => [
            'notEmpty',
            'string'
        ],
        'intFromPhpDoc'          => [
            'notEmpty',
            'integer'
        ],
        'noTypeWithFloatDefault' => [
            'notEmpty',
            'float'
        ]
    ];

    protected const SETTERS = [];
}
```

### 创建任务处理器
```bash
$ php app.php create:jobHandler <name>
```

此命令会创建名为 `<Name>Job` 的类。

#### 示例：
```bash
$ php app.php create:jobHandler my
```

生成的类（`<app directory>/src/Job/MyJob.php`）代码如下：
```php
use Spiral\Jobs\JobHandler;

class MyJob extends JobHandler
{
    public function invoke(): void
    {
    }
}
```

###  创建中间件
```bash
$ php app.php create:middleware <name>
```

此命令会创建名为 `<Name>` 的类。

#### 示例：
```bash
$ php app.php create:middleware my
```

生成的类（`<app directory>/src/Middleware/My.php`）代码如下：
```php
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;

class My implements MiddlewareInterface
{
    /**
     * {@inheritdoc}
     */
    public function process(Request $request, RequestHandlerInterface $handler): Response
    {
        return $handler->handle($request);
    }
}
```

### 创建数据库迁移
```bash
$ php app.php create:migration <name>
```

此命令会生成名为 `<Name>Migration` 的类。可用选项：

* `--table (-t)` 指定数据库表名
* `--column (-c)` -（可以多次使用） 指定列定义。指定表名时才可用。列格式是 `name:type`.

>  关于数据库迁移的更多信息，请查看 [migrations](https://github.com/spiral/migrations) 组件包

#### 示例
```bash
$ php app.php create:migration my
```

生成的类（`<app directory>/src/migrations/<date>.<time>-<index>-my.php`）代码如下：

```php
use Spiral\Migrations\Migration;

class MyMigration extends Migration
{
    /**
     * Create tables, add columns or insert data here
     */
    public function up(): void
    {
    }

    /**
     * Drop created, columns and etc here
     */
    public function down(): void
    {
    }
}
```

#### 待选项的示例

```bash
$ php app.php create:migration my -t my_table -c int_col:int
```

生成的类（`<app directory>/migrations/<date>.<time>-<index>-my.php`）代码如下：

```php
use Spiral\Migrations\Migration;

class MyMigration extends Migration
{
    /**
     * Create tables, add columns or insert data here
     */
    public function up(): void
    {
        $this->table('my_table')
            ->addColumn('int_col', 'int')
            ->create();
    }

    /**
     * Drop created, columns and etc here
     */
    public function down(): void
    {
        $this->table('my_table')->drop();
    }
}
```

### 创建实体仓库

```bash
$ php app.php create:repository <name>
```

此命令会创建名为 `<Name>Repository` 的类。

#### 示例：
```bash
$ php app.php create:repository my
```

生成的类（`<app directory>/src/Repository/MyRepository.php`）代码如下：

```php
use Cycle\ORM\Select\Repository;

class MyRepository extends Repository
{
}
```

### 创建实体

```bash
$ php app.php create:entity <name> [<format>]
```

此命令会创建名为 `<Name>Entity` 的类（译注：实测生成的类名是 `<Name>`）。

`format` 负责声明格式, 当前只支持 [annotations](https://github.com/cycle/annotated) 格式。可用选项：

* `--role (-r)` - 实体角色，默认为不含命名空间的类名称的小写格式
* `--mapper (-m)` - 映射器类名，默认是 `Cycle\ORM\Mapper\Mapper`
* `--table (-t)` - 实体的源表，默认是实体角色的复数形式
* `--accessibility (-a)` - 属性可访问性（public, protected, private）, 默认为 public
* `--inflection (-i)` - 列名变形选项，可用的值有：tableize (t), camelize (c). 参见 [Doctrine inflector](https://github.com/doctrine/inflector)
* `--field (-f)` -（允许多次使用）添加一个列，格式为 "name:type"
* `--repository (-e)` - 用于实体的数据读取操作的数据仓库类，默认是 `Cycle\ORM\Select\Repository`
* `--database (-d)` - 数据库连接，默认是 null（使用 default 连接）

#### 示例：
```bash
$ php app.php create:entity my
```

生成的类（`<app directory>/src/Database/My.php`）代码如下：
```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
}
```

#### 指定可访问性为 public 的示例：
```bash
$ php app.php create:entity my -f field:string
```

生成的类（`<app directory>/src/Database/My.php`）代码如下：
```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
    /**
     * @Cycle\Column(type = "string")
     */
    public $field;
}
```

#### 指定可访问性为 protected/private 的示例：
```bash
$ php app.php create:entity my -f field:string -a protected
```

生成的类（`<app directory>/src/Database/My.php`）代码如下：
```php

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
    /**
     * @Cycle\Column(type = "string")
     */
    protected $field;

    public function setField(string $value)
    {
        $this->field = $value;
    }

    public function getField()
    {
        return $this->field;
    }
}
```

#### 使用 tableize 变形选项的示例：
```bash
$ php app.php create:entity my -f int_field:int -f stringField:string -i t
```

生成的类（`<app directory>/src/Database/My.php`）代码如下：
```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
    /**
     * @Cycle\Column(type = "int")
     */
    public $int_field;

    /**
     * @Cycle\Column(type = "string", name = "string_field")
     */
    public $stringField;
}
```

#### 使用 camelize 变形选项的示例：
```bash
$ php app.php create:entity my -f int_field:int -f stringField:string -i c
```

生成的类（`<app directory>/src/Database/My.php`）代码如下：
```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class My
{
    /**
     * @Cycle\Column(type = "int", name = "intField")
     */
    public $int_field;

    /**
     * @Cycle\Column(type = "string")
     */
    public $stringField;
}
```

#### 其它选项示例：

```bash
$ php app.php create:entity my -r myRole -m MyMapper -t my_table -d my_db -e
```

生成的类（`<app directory>/src/Database/My.php`）代码如下：

```php
use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity(role = "myRole", mapper = "MyMapper", repository = "my", table = "my_table", database = "my_db")
 */
class My
{
}
```

同时会生成数据仓库类 (`<app directory>/src/Repository/MyRepository.php`):

```php
use Cycle\ORM\Select\Repository;

class MyRepository extends Repository
{
}
```
