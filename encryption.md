# Encryption
Spiral provides a simple core component used to perform basic data encyption. The component is based 
on PHP OpenSSL library and is used to encrypt Http Cookies. If you want to use it in your code,
check following examples.

To get component instance or facade read following topics: [Core Components] (core/bindings.md), 
[Facades] (core/facades.md). All provided examples assume a controller action where the component
was created via string shortcut.

## Configuring
Before using encrypter you have to make sure that the correct encrypting password (key) is specified. 
You can use the key manually:

```php
$this->encrypter->setKey('LKHasd0862l;+)9s');
```
> The key should be 16, 24 or 32 characters long ( for 128, 192 and 256 bit keys respectively).

By default, spiral will use the common application key which is specified in the `config/encrypter.php` config file. 
The common application key is updated at time of application creation, and can be altered at any moment
manually or via running console command `./spiral.cli core:key`.

## Encrypting
To encrypt simple string or array run

```php
$encrypted = $this->encrypter->encrypt('some data');
```

Decrypting can be done using:

```php
$data = $this->encrypter->decrypt($encrypted);
```

> Attention, in many cases you can get DecryptionException (bad data, wrong key and etc), don't forget to handle it.

## Encrypting method (cipher)
By default spiral uses `aes-256-cbc` as the cipher method, this value can be changed globally by altering the encrypter config, 
or locally by executing:

```php
$this->encrypter->setMethod('des-ecb');
```
> To get all available methods execute [`openssl_get_cipher_methods`] (http://php.net/manual/en/function.openssl-get-cipher-methods.php).
