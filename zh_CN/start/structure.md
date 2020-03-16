# 应用程序结构
Spiral 框架对命名空间和目录结构不做强制要求，你可以根据自己的喜好和习惯来组织它们。

## 目录结构
目录结构可以通过内核类的 `mapDirectories` 方法来控制。默认状态下，所有的应用相关目录都按照以下模式基于 `root` 来计算：

目录 | 值 
---       | ---
root      | **用户指定**
vendor    | **root**/vendor
runtime   | **root**/runtime
cache     | **root**/runtime/cache
public    | **root**/public
app       | **root**/app
config    | **app**/config
resources | **app**/resources

有些组件会定义它们自己的目录，例如：

组件         | 目录  | 值 
---               | ---        | ---
spiral/views      | views      | **app**/views
spiral/translator | locale     | **app**/locale
spiral/migrations | migrations | **app**/migrations

## 初始目录
你可以在你的 `app.php` 文件中，通过给 `App` 的 `init` 方法传入参数来指定你的目录：

```php
$app = \App\App::init([
    'root' => __DIR__
]);
```

比如我们要改变 `runtime` 目录的位置：

```php
$app = \App\App::init([
    'root' => __DIR__, 
    'runtime' => sys_get_temp_dir()
]);
```

如果你在程序代码中要解析目录别名，可以使用 `Spiral\Boot\DirectoriesInterface`：

```php
function test(DirectoriesInterface $dirs)
{
    dump($dirs->get('app'));
    // <your root dir>/app/
}
```

还可以在全局 Ioc 作用域（配置文件、控制器、服务的代码中）下使用 `directory` 方法：

```php
function test()
{
    dump(directory('app'));
    // <your root dir>/app/
}
```

## 命名空间
默认情况下，所有的应用程序框架（Web, CLI, GRPC）都使用 `App` 根命名空间指向 `app/src` 目录。你可以在 `composer.json` 文件中按自己的需求进行修改：

```json
{
  "autoload": {
    "psr-4": {
      "App\\": "app/src/"
    }
  }
}
```
