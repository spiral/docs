# 代码生成

`spiral/reactor` 组件提供了流畅的声明式包装器用于生成 PHP 类代码。该组件是 `spiral/scaffolder`（脚手架）组件的基础。但它也可以在脚手架之外单独使用。比如开发者需要在自己的框架中扩展脚手架功能。

安装：

```bash
$ composer require spiral/reactor
``` 

## 类定义

可以使用 `Spiral\Reactor\ClassDeclaration` 来声明类。

```php
use Spiral\Reactor\ClassDeclaration;

$class = new ClassDeclaration('MyClass');

dump($class->render());
```

输出如下：

```php
class MyClass
{
}
```

你可以通过生成的类直接访问或声明它的大多数成员：

### 属性

声明类的属性：

```php
$class = new ClassDeclaration('MyClass');

$class->property('my_property')
    ->setProtected()
    ->setDefaultValue('default')
    ->setComment(['示例属性', '@var string']);

dump($class->render());
```

输出如下：

```php
class MyClass
{
    /**
     * 示例属性
     * @var string
     */
    protected $my_property = 'default';
}
```

### 常量

声明类的常量：

```php
$class = new ClassDeclaration('MyClass');

$class->constant('MY_CONSTANT')
    ->setPublic()
    ->setValue('default')
    ->setComment(['示例常量']);

dump($class->render());
```

输出如下：

```php
class MyClass
{
    /**
     * 示例常量
     */
    public const MY_CONSTANT = 'default';
}
```

### 特性（Trait）

给类添加 trait:

```php
$class = new ClassDeclaration('MyClass');

$class->addTrait(PrototypeTrait::class);

dump($class->render());
```

输出如下：

```php
class MyClass
{
    use Spiral\Prototype\Traits\PrototypeTrait;
}
```

### 接口和扩展

实现指定的接口，或者扩展基础类：

```php
use Spiral\Reactor\ClassDeclaration;
use Cycle\ORM\Select;

$class = new ClassDeclaration('MyClass');

$class->addInterface(\Countable::class)->setExtends(Select\Repository::class);

dump($class->render());
```

输出如下：

```php
class MyClass extends Cycle\ORM\Select\Repository implements Countable
{
}
```

### 方法

生成类的方法：

```php
$class = new ClassDeclaration('MyClass');

$method = $class->method('ping')
    ->setPublic()
    ->setComment('示例方法')
    ->setReturn('void');

$method->parameter('a')->setType('string')->setDefaultValue('test');
$method->setSource('echo $a');

dump($class->render());
```

输出如下：

```php
class MyClass
{
    /**
     * 示例方法
     */
    public function ping(string $a = 'test'): void
    {
        echo $a;
    }
}
```

## 命名空间

把生成的类归入指定的命名空间：

```php
use Spiral\Reactor\ClassDeclaration;
use Spiral\Reactor\NamespaceDeclaration;

$class = new ClassDeclaration('MyClass');

$ns = new NamespaceDeclaration('MyNamespace');
$ns->addElement($class);

dump($ns->render());
```

输出如下：

```php
namespace MyNamespace {
    class MyClass
    {
    }
}
```

## 文件

或者，你可以通过代码文件定义来定义整个 PHP 文件（指定命名空间）：

```php
use Spiral\Reactor\ClassDeclaration;
use Spiral\Reactor\FileDeclaration;
use Cycle\ORM\Select;

$class = new ClassDeclaration('MyClass');

$file = new FileDeclaration('MyNamespace');

$file->setComment('示例文件');
$file->setDirectives('strict_types=1');

$file->addUse(Select\Repository::class, 'Repo');

$file->addElement($class);

dump($file->render());
```

输出如下：

```php
<?php
/**
 * 示例文件
 */
declare(strict_types=1);
namespace MyNamespace;

use Cycle\ORM\Select\Repository as Repo;

class MyClass
{
}
```
