# 框架 - 终结器

绝大部分的框架组件在请求结束后不需要进行任何重置操作。但还是会有某些应用场景，可能会需要在用户请求完成之后做些重置工作。

> 请优先使用 IoC 作用域而不是终结器。

## 终结器接口

要实现终结器，需要继承 `Spiral\Boot\FinalizerInterface` 接口：

```php
/**
 * 用于关闭长时间运行的进程中的资源或者连接
 */
interface FinalizerInterface
{
    /**
     * 终结器会在每个请求结束之后执行，用于内存回收或关闭打开的连接。
     *
     * @param callable $finalizer
     */
    public function addFinalizer(callable $finalizer);
    
    /**
     * 终结器的执行函数
     *
     * @param bool $terminate 如果终结器会结束整个应用程序，设置为 true
     */
    public function finalize(bool $terminate = false);
}
```

终结器可以被所有类型的应用程序调度器调用：

* HTTP 请求结束时
* HTTP 请求报错失败时
* 任务（job）完成时
* 任务（job）报错失败时
* GRPC 调用结束时
* GRPC 调用报错失败时
* 控制台命令完成时

> 注意：终结器只在指定的调度器成功启动之后才会被调用。因此在终结器中可以自由调用应用程序命令和 HTTP 方法，无需直接使用使用调度器，也无需在每个请求结束后重置服务。

传入终结器的处理程序（`finalize` 方法）的第一个布尔值参数代表应用程序是否会在当前请求之后终止。

> 注意：避免在终结器中重置 IoC 设置，这可能会导致某些单例服务缓存先前的服务版本。

## 终结器示例

我们用一个示例来演示如何在每次请求之后自动关闭数据库连接。在应用程序运行很多工作进程（或者 lambda 函数）而不想耗尽所有的数据库套接字（socket）时，这个功能会很有用。

```php
// 在引导程序中
/**
 * @param FinalizerInterface $finalizer
 * @param ContainerInterface $container
 */
public function boot(FinalizerInterface $finalizer, ContainerInterface $container)
{
    $finalizer->addFinalizer(function () use ($container) {
        /** @var DatabaseManager $dbal */
        $dbal = $container->get(DatabaseManager::class);
 
        foreach ($dbal->getDrivers() as $driver) {
            $driver->disconnect();
        }
    });
}
```

> 示例中的这个引导程序已经包含在 Spiral 框架中的 `Spiral\Bootloader\Database\DisconnectsBootloader` 这个引导程序内。
