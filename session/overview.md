# Session
Spiral session implementation based on native PHP session mechanism with few extensions.

Session provides basic security mechanism using user signature and represent session data via set of isolated [session sections/segments](/session/usage.md).
 
## Initiating Session
Session cookie control is handled internally in order to unify cookie management and allow CLI based application testing.

In order to enable session support in your application add `Session\Http\SessionStarter` middleware into http config or route.

Session will be started on demand when data read is requested.

## Session Options
Alter your session configuration using config file "session":

```php
return [
    /*
     * Default session lifetime is 1 day.
     */
    'lifetime' => 86400,

    /*
     * Cookie name for sessions. Used by SessionStarter middleware. Other cookies options will
     * be gathered from HttpConfig. You can combine SessionStarter with CookieManager in order
     * to protect your cookies.
     */
    'cookie'   => env('SESSION_COOKIE', 'SID'),

    /*
     * Default handler is set to file. You can switch this values based on your environments.
     * SessionStore will be initiated on demand to prevent performance issues. Since spiral provides
     * set of widgets to with html forms over ajax sessions are mainly used to store authorization
     * data and not used to flush errors at page.
     *
     * You can set this value to "native" to disable custom session handler and use default php
     * mechanism.
     */
    'handler'  => env('SESSION_HANDLER', 'files'),

    /*
     * Session handler. You are able to use bind() function in handler options.
     */
    'handlers' => [
        //Debug session handler without ability to save anything
        'null'  => [
            'class' => Handlers\NullHandler::class
        ],
        //File based session
        'files' => [
            'class'   => Handlers\FileHandler::class,
            'options' => [
                'directory' => directory('runtime') . 'sessions'
            ]
        ],
        /*{{handlers}}*/
    ]
];
```

## Session Handlers
You can use `SessionHandlerInterface` compatible adapters to make your session work with custom storage. Keep handler value `null` in order to native PHP session handler.

> All handlers are resolved via container, feel free to define custom dependencies and config values in your implementations.