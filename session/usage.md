# Working with Session
On practice spiral session work as simple facade at top of native php _SESSION implementation. Differences include bypassing default cookie control and splitting session into segments/sections.

## Session Access
You are able to access active session instance inside your controllers and services using shortcut "session" or `SessionInterface` dependency:

```php
public function indexAction(SessionInterface $session)
{
    assert($this->session == $session);
}
```

> Please note that session access will only work inside SessionStarter middleware which is enabled in your application by default.

### Start Session
You are able to control session flow using set of methods defined in `SessionInterface`:

```php
interface SessionInterface extends InjectorInterface
{
    /**
     * @return bool
     */
    public function isStarted(): bool;

    /**
     * Resume session or start new one.
     *
     * @throws \Spiral\Session\Exceptions\SessionException
     */
    public function resume();

    /**
     * Current session ID. Null when session is destroyed.
     *
     * @return null|string
     */
    public function getID();

    /**
     * Regenerate session id without altering it's data.
     *
     * @return self
     */
    public function regenerateID(): self;

    /**
     * Commit session data, must return true if data successfully saved.
     *
     * @return bool
     */
    public function commit(): bool;

    /**
     * Destroys all data associated with session but does not regenerate it IDs.
     *
     * @return bool
     */
    public function destroy(): bool;

    /**
     * @param string|null $name When null default section to be returned.
     *
     * @return SectionInterface
     */
    public function getSection(string $name = null): SectionInterface;
}
```

> Read how to control session cookies [here](/session/overview.md).

## Session Data
Due session is being based on native php implementation you can freely use _SESSION global variable in your legacy code, though you are required to start session manually in this case:

```php
public function indexAction()
{
    $this->session->resume();

    if (!isset($_SESSION['value'])) {
        $_SESSION['value'] = 0;
    }
    
    dump($_SESSION['value']++);
}
```

## Session Sections
More convenient way of working with session involves `SectionInterface` which isolated all of it's from the rest of the session.

You can get session segment using `getSection` method of `SessionInterface` or via contextual injection.

```php
public function indexAction(SectionInterface $mySession)
{
    dump($mySession->getName()); //mySession

    dump($this->session->getSection('mySession'));
}
```

`SegmentInterface` adds set of methods to simplify data access:

```php
interface SectionInterface extends \IteratorAggregate, \ArrayAccess
{
    /**
     * Section name.
     *
     * @return string
     */
    public function getName(): string;

    /**
     * All section data in a form of array.
     *
     * @return array
     */
    public function all(): array;

    /**
     * Set data in session.
     *
     * @param string $name
     * @param mixed  $value
     *
     * @return mixed
     * @throws SessionException
     */
    public function set(string $name, $value);

    /**
     * Check if value presented in session.
     *
     * @param string $name
     *
     * @return bool
     * @throws SessionException
     */
    public function has(string $name);

    /**
     * Get value stored in session.
     *
     * @param string $name
     * @param mixed  $default
     *
     * @return mixed
     * @throws SessionException
     */
    public function get(string $name, $default = null);

    /**
     * Read item from session and delete it after.
     *
     * @param string $name
     * @param mixed  $default Default value when no such item exists.
     *
     * @return mixed
     * @throws SessionException
     */
    public function pull(string $name, $default = null);

    /**
     * Delete data from session.
     *
     * @param string $name
     *
     * @throws SessionException
     */
    public function delete(string $name);

    /**
     * Clear all session section data.
     */
    public function clear();
}
```

> Sections will resume session automatically on demand.

## Contextual Injections
Spiral Session implementation support contextual injections based on parameter name, this allow you to define your services and models like that:

```php
class Model
{
    private $session;

    public function __construct(SessionInterface $modelSession)
    {
        $this->session = $modelSession;
    }
}
```

> All sections/segments are fetched from active session instance.
