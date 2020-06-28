# 复合过滤器

过滤器组件还提供了创建嵌套过滤器和嵌套的过滤器数组的能力。为了演示这一特性，我们使用一个简单的过滤器：

```php
class AddressFilter extends Filter
{
    protected const SCHEMA = [
        'city'    => 'data:city',
        'address' => 'data:address'
    ];
}
```

上面的过滤器接受的数据格式如下：

```json
{
  "city": "旧金山",
  "address": "详细街道地址"
}
```

## 子过滤器

你可以把在过滤器中嵌套使用其它的过滤器，从而创建复合过滤器。只要把字段的源指定为嵌套的过滤器类即可：

```php
class ProfileFilter extends Filter
{
    protected const SCHEMA = [
        'name'    => 'data:name',
        'address' => AddressFilter::class
    ];
}
```

上面的过滤器接受的数据格式如下：

```json
{
  "name": "安东尼",
  "address": {
    "city": "旧金山",
    "address": "具体地址"
  }
}
```

可以通过 `getField` 方法或者魔术方法 `__get` 来访问嵌套的过滤器：

```php
public function index(ProfileFilter $p)
{
    dump($p->address->city); // 旧金山
}
```

执行验证时，两个过滤器会一起验证。如果在 `address` 过滤器中发生错误，该错误会挂载到子数组中：

```json
{
  "name": "This field is required.",
  "address": {
    "city": "This field is required."
  }
}
```

### 自定义前缀

某些情况下，你可能需要使用与实际分配给嵌套过滤器的键（名称）不同的数据前缀，这种情况可以使用数组形式，数组中的第一个元素是过滤器类名，第二个元素是数据前缀：

```php
class ProfileFilter extends Filter
{
    protected const SCHEMA = [
        'name'    => 'data:name',
        'address' => [AddressFilter::class, 'addr']
    ];
}
```

这个过滤器接受的数据格式如下：

```json
{
  "name": "This field is required.",
  "addr": {
    "city": "This field is required."
  }
}
```

> 你可以在内部跳过使用 `address` 键名，错误会对应地进行挂载。

## 过滤器数组

你可以同时填充一组过滤器。将单个字段指向的嵌套过滤器声明为数组即可：

```php
class MultipleAddressesFilter extends Filter
{
    protected const SCHEMA = [
        'key'       => 'data:key',
        'addresses' => [AddressFilter::class]
    ];
}
```

这个过滤器接受的数据格式如下：

```json
{
  "key": "value",
  "addresses": [
    {
      "city": "San Francisco",
      "address": "Address"
    },
    {
      "city": "Minsk",
      "address": "Address #2"
    }
  ]
}
```

访问数据时可以使用数组访问器来访问对应的数据：

```php
public function index(MultipleAddressesFilter $ma)
{
    dump($ma->addresses[0]->city); // San Francisco
    dump($ma->addresses[1]->city); // Minsk
}
```

> 过滤器数组产生的错误会对应地挂载。

### 自定义前缀

如果在使用过滤器数组的同时还想自定义前缀，方法与之前一样，不同的是键名必须加上 `.*`，以表示这是一个数组前缀：

```php
class MultipleAddressesFilter extends Filter
{
    protected const SCHEMA = [
        'key'       => 'data:key',
        'addresses' => [AddressFilter::class, 'addr.*']
    ];
}
```

这个过滤器接受的数据格式如下：

```json
{
  "key": "value",
  "addr": [
    {
      "city": "San Francisco",
      "address": "Address"
    },
    {
      "city": "Minsk",
      "address": "Address #2"
    }
  ]
}
```

在这种情况下，你依然可以使用 `address` 键来访问嵌套的数组数据：

```php
public function index(MultipleAddressesFilter $ma)
{
    dump($ma->getField('addresses')[0]->getField('city')); // San Francisco
    dump($ma->getField('addresses')[1]->getField('city')); // Minsk
}
```

## 符合过滤器

你可以使用嵌套的子过滤器作为一个更复杂的符合过滤器的一部分，但不需要嵌套实际数据。方法是使用 `.` 作为前缀：

```php
class CompositeFilter extends Filter
{
    protected const SCHEMA = [
        'name'    => 'data:name',
        'address' => [AddressFilter::class, '.'],
    ];
}
```

这样一来，`AddressFilter` 会直接对顶级数据进行处理，因此可以发送这样的请求数据：

```json
{
  "name": "Antony",
  "city": "San Francisco",
  "address": "Address"
}
```
