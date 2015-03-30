# Session
Spiral session component based on default php session handles with only one general change which disables automatic cookie creation (this operation handle via `SessionStarter` http middleware).

By default session component (`SessionStore`) configured to work with
default php session handler, however component provides set of custom handlers: Memcache, Redis and File. You can add custom or external handler any moment by implementing `SessionHandlerInterface`.

> Session component handlers configuration located in `session.php` config file.

## Starting Session
Spiral will automatically start session for you at your first request for session data, this may speedup your application a lot as no handler should be initiated before you need it. 

Hovewer you can control session flow manually by operating low level functions like `commit()`, `setID()` and etc.

EXAMPLES

> Attention, you can't use `setID` method if session already started you should either stop it previously or use `regenerateID` method.

## Using Session
SessionStore implements spiral singleton pattern and can be used via DI, shortcut binding (`spiral`) in controllers or via static facade.

EXAMPLES
 