# Pagination
Spiral framework provides simple interface to expose pagination methods `limit` and `offset`. Such methods are implemented
in DBAL Select Query builder and Cycle Select objects.

You can implement this interface in any of your classes as well:

```php
interface PaginableInterface
{
    /**
     * Set the pagination limit.
     *
     * @param int $limit
     */
    public function limit(int $limit);

    /**
     * Set the pagination offset.
     *
     * @param int $offset
     */
    public function offset(int $offset);
}
```
 
## Pagination
To paginate your query you must object an pagination object first. Since pagination depends on current request scope
you must either create this object by yourself or use factory `Spiral\Pagination\PaginationProviderInterface`:

```php
public function index(PaginationProviderInterface $paginators)
{
    $users = $this->users->select();

    $paginators->createPaginator('page')->paginate($users);

    dump($users->fetchAll());
}
```

This factory is enabled in your Web application by default via `Spiral\Bootloader\Http\PaginationBootloader`. The first
argument passed to `createPaginator` method is name of query parameter which used to provide page number.

> Technically, you can use pagination in any dispatcher, page parameter can be provided by any user payload.

If you use `spiral/prototype` extension you can also access `PaginationProviderInterface` via `paginators` shortcut:

```php
public function index()
{
    $users = $this->users->select();

    $this->paginators->createPaginator('page')->paginate($users);

    dump($users->fetchAll());
}
```

## Using Paginator Object
Paginator object provided by the factory does not implement any other method rather than `paginate`:

```php
interface PaginatorInterface
{
    /**
     * Paginate the target selection and return new paginator instance.
     *
     * @param PaginableInterface $target
     * @return PaginatorInterface
     */
    public function paginate(PaginableInterface $target): PaginatorInterface;
}
```

However, default implementation `Spiral\Pagination\Paginator` has a number of useful methods which can help in response
rendering:

```php
public function index()
{
    $users = $this->users->select();

    $paginator = $this->paginators->createPaginator('page', 50)->paginate($users); // limit 50

    dump($users->fetchAll());

    assert($paginator instanceof Paginator);

    dump($paginator->countDisplayed()); // always 50 except last page
    dump($paginator->getPage()); // current page
    dump($paginator->getParameter()); // page parameter (for Uri building)
    dump($paginator->isRequired()); // false when only 1 page is presented
}
```