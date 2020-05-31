# 安全 - 数据加密

Web 和 GRPC 应用程序模板都默认包含加密组件。要在其它版本中安装加密组件，可以执行：

```bash
$ composer require spiral/encrypter
```

安装之后必须在注册 `Spiral\Bootloader\Security\EncrypterBootloader` 引导程序来启用该组件：

## 应用程序密钥

加密组件基于 [defuse/php-encryption](https://github.com/defuse/php-encryption) 实现。它要求应用程序提供加密所需的密钥：

默认情况下 `EncrypterBootloader` 引导程序会从环境变量 `ENCRYPTER_KEY` 中读取采用 Base64 编码的密钥。

如果使用了 [Dotenv](/zh_CN/extension/dotenv.md) 扩展，则可以应用程序根目录下的 `.env` 文件中指定该密钥的值。也可以通过控制台命令来生成一个新的密钥：

```bash
$ php app.php encrypt:key -m .env
```

> 请注意，加密组件会用于保护 cookie 值，因此修改该密钥值会使所有之前的 cookie 全部失效。

## 使用

在应用程序中可以通过 `Spiral\Encrypter\EncrypterInterface` 来使用加密组件：

```php
/**
 * 用于加密服务的不可变类
 */
interface EncrypterInterface
{
    /**
     * 用指定的 key 创建加密器实例
     *
     * @param string $key
     * @return self
     *
     * @throws EncrypterException
     */
    public function withKey(string $key): EncrypterInterface;

    /**
     * 加密器的密钥值。以 ANSI 字符串格式返回
     *
     * @return string
     */
    public function getKey(): string;

    /**
     * 把数据加密为字符串值。只能通过相同密钥的加密器实例的 decrypt() 方法解密
     *
     * @param mixed $data
     * @return string
     *
     * @throws EncryptException
     * @throws EncrypterException
     */
    public function encrypt($data): string;

    /**
     * 对加密字符串进行解密。加密字符串必须是由相同密钥的加密器实例的 encrypt() 方法加密所得
     *
     * @param string $payload
     * @return mixed
     *
     * @throws DecryptException
     * @throws EncrypterException
     */
    public function decrypt(string $payload);
}
```

加密器除了通过 `EncrypterInterface` 之外，也可以通过[开发原型属性](/zh_CN/basics/prototype.md) `encrypter` 来访问：

```php
protected function index(EncrypterInterface $encrypter)
{
    $payload = $encrypter->encrypt(['abc']);
    dump($payload);

    dump($this->encrypter->decrypt($payload));
}
```
