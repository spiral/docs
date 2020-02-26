# 框架 - 静态高速缓存

Spiral 框架通过 `spiral/boot` 组件提供了一些很方便的接口，用来存储在进程之间共享的数据。

> 静态缓存的当前实现方式是借助 OpCache 把数据存储在物理文件中。但未来的实现方式会把数据存储迁移到 RoadRunner 或与 SHM 共享的 PHP 扩展中。因此请注意不要在你的代码中把数据与目前实现方式下的物理文件关联起来。

## 缓存接口

在项目中，可以利用 `Spiral\Boot\MemoryInterface` 接口来存储结算结果：

```php
/**
 * 内存高速缓存。借助此接口来存储计算结果，但千万不要用其存储非静态数据或用户数据（!）。
 */
interface MemoryInterface
{
    /**
     * 从内存中的长缓存读取数据。必须返回与存储时一致的值，或者 null. 
     * 当前约定只允许存储可序列化（var_export-able）的数据
     *
     * @param string $section 不区分大小写
     * @return string|array|null
     */
    public function loadData(string $section);

    /**
     * 向内存中的长缓存存入数据。不支持内部引用或者闭包。
     * 当前约定只允许存储可序列化（var_export-able）的数据
     *
     * @param string       $section 不区分大小写
     * @param string|array $data
     */
    public function saveData(string $section, $data);
}
```

## 用例

关于内存的一般想法是通过缓存某些功能的执行结果，以提升应用程序的速度。高速缓存组件通常用来存储配置缓存、ORM 和 ODM 模式（schema），控制台命令列表和令牌生成器缓存；也可以用来缓存预编译的路由等。
 
 > 应用缓存永远不可用于存储用户数据。

## 示范实例

接下来看一个服务示例，这个服务用于分析计算某些行为（或操作）的可用类：

```php
abstract class Operation 
{
    /**
     * 执行某种操作
     */
    abstract public function perform($request);
}

class OperationService
{
    /**
     * 可用操作与类的关系列表
     */
    protected $operations = [];

    /**
     * 操作服务构造函数
     *
     * @param MemoryInterface  $memory
     * @param ClassesInterface $classes
     */
    public function __construct(MemoryInterface $memory, ClassesInterface $classes)
    {
        $this->operations = $memory->loadData('operations');

        if (is_null($this->operations)) {
            $this->operations = $this->locateOperations($classes); // 这是一个很耗时的操作
            $memory->saveData('operations', $this->operations);
        }      
    }

    /**
     * @param string $operation
     * @param mixed  $request
     */
    public function run($operation, $request)
    {
        // 基于 $operations 属性来执行具体操作
    }

    /**
     * @param ClassesInterface $locator
     * @return array
     */
    protected function locateOperations(ClassesInterface $classes)
    {
        // 通过扫描每个可用类，生成可用操作的列表
    }
}
```

> 目前只能在高速缓存中存储数组或者标量值

通过实现 `Spiral\Boot\MemoryInterface` 接口，你可以创建自己的缓存管理服务，改成用 APC、XCache、RoadRunner 的 DHT、Redis 甚至是 Memcache 来实现。

在将 `Spiral\Boot\MemoryInterface` 嵌入到组件或者服务之前，要注意：

* 高速缓存只能用于逻辑缓存，不要将任何与用户请求相关的数据存入其中
* 要假设高速缓存中的数据随时可能消失（非持久化）
* `saveData` 是线程安全的方法，但是在高并发时性能会下降
* `saveData` 比 `loadData` 成本更高，不要程序运行时向高速缓存进行写入
* 使用高速缓存的最佳场合是引导程序或者控制台命令
