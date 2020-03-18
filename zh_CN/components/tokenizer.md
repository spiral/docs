# 静态分析

很多 Spiral 组件都基于自动代码分析和发现功能。其中最常用的功能是查找类声明，由 `Spiral\Tokenizer\ClassesInterface` 提供。

> Tokenizer 组件在官方提供的所有应用模板项目中都已经预装了。

## 类定位器

使用 `Spiral\Tokenizer\ClassesInterface` 可以通过类的名称、接口或者 trait 找到可用的类：

```php
public function findClasses(ClassesInterface $classes)
{
    foreach ($classes->getClasses(\Psr\Http\Server\MiddlewareInterface::class) as $middleware) {
        dump($middleware->getFileName());
    }
}
```

默认情况下，该组件只从 `app`  目录下查找可用类。当然你可以借助 `Spiral\Bootloader\TokenizerBootloader` 自己添加其他的查找目录。

```php
public function boot(TokenizerBootloader $tokenizer)
{
    $tokenizer->addDirectory(directory('vendor') . 'name/extension/src');
}
```

> 提示：查找类的过程速度有些慢，请谨慎使用，建议只添加非常必要的目录。

## PHP 解析器

默认情况下，在 Web 和 GRPC 项目中可以使用 [nikic/PHP-Parser](https://github.com/nikic/PHP-Parser). 通过这个依赖项可以进行更深度的 AST 树分析。

## ORM 自查

获取所有可用的实体角色列表：

```php
use Cycle\ORM\ORMInterface;

// ...

public function index(ORMInterface $orm)
{
    dump($orm->getSchema()->getRoles());
}
```

获取所有 ORM 实体类：

```php
use Cycle\ORM\ORMInterface;
use Cycle\ORM\Schema;

// ...

public function index(ORMInterface $orm)
{
    foreach($orm->getSchema()->getRoles() as $role) {
        dump($orm->getSchema()->define($role, Schema::ENTITY));
    }
}
```
