# Encryption
Spiral provides simple core component used to perform basic data encyption. Component based 
on PHP OpenSSL library and used to encrypt Http Cookies. If you want to use it in your code,
check following examples.

To get component instance or facade read following topics: [Core Components] (core/bindings.md), 
[Facades] (core/facades.md). All provided examples given for controller action where component
received via string binding.

## Configuring
Before using encrypter you have to make sure that correct encrypting password (key) is specified. 
You can force such key manually:
```php
$this->encrypter->setKey('LKHasd0862l;+)9s');
```
> Key should be 16, 24 or 32 characters long (accordingly to 128, 192 and 256 bit keys).

By default spiral will use common application key which is speicfied `config/encrypter.php` config file. 
Common application key is updated at time of application creation, and can be altered at any moment
manually or via running console command `./spiral.cli core:key`.

## Encrypting
To encrypt simple string or array runL
```php
$encrypted = $this->encrypter->encrypt('some data');
```
Decrypting can be done using:
```php
$data = $this->encrypter->decrypt($encrypted);
```

> Attention, in many cases you can get DecryptionException (bad data, wrong key and etc), don't forget to handle it.

## Encrypting method (cipher)
By default spiral use `aes-256-cbc` as cipher method, this value can be changed globally by altering encrypter config, 
or locally by executing:
```php
$this->encrypter->setMethod('des-ecb');
```
> To get all available methods execute [`openssl_get_cipher_methods`] (http://php.net/manual/en/function.openssl-get-cipher-methods.php).
