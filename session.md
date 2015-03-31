# Session
The spiral session component is based on the default php session handles with only one general change which disables automatic cookie creation (this operation handle via `SessionStarter` http middleware).

By default the session component (`SessionStore`) is configured to work with the
default php session handler, however, the component provides a set of custom handlers: Memcache, Redis and File. You can add custom or external handler at any time by implementing `SessionHandlerInterface`.

> The session component handlers' configuration is located in the `session.php` config file.

## Starting Session
Spiral will automatically start session for you at your first request for session data, this may speedup your application a lot as no handler should be initiated before you need it. 

Hovewer you can control session flow manually by operating low level functions like `commit()`, `setID()` and etc.

EXAMPLES

> Attention, you can't use `setID` method if the session is already started; you should either stop it beforehand or use the `regenerateID` method.

## Using Session
SessionStore implements the spiral singleton pattern and can be used via DI, shortcut binding (`spiral`) in controllers or via static facade.

EXAMPLES
 
