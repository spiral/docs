# Pagination
Pagination is one of the most imporant part of any project interface. Spiral trying to clearly separate pagination process by itself and way used to represent pagination in template or page.

## Paginable Objects
If you wish to apply pagination to your data source/object, you only have to implement `PaginableInterface` which requires few methods to be defined. In addition to that, object must expose `count` method (however this method might not be used by countless paginators):

```php
interface PaginableInterface extends \Countable
{
    /**
     * Set selection limit.
     *
     * @param int $limit
     * @return mixed
     */
    public function limit($limit = 0);

    /**
     * Set selection offset.
     *
     * @param int $offset
     * @return mixed
     */
    public function offset($offset = 0);

    /**
     * Manually set paginator instance for specific object.
     *
     * @param PaginatorInterface $paginator
     * @return $this
     */
    public function setPaginator(PaginatorInterface $paginator);

    /**
     * Indication that object was paginated.
     *
     * @return bool
     */
    public function isPaginated();

    /**
     * Get paginator for the current selection. Paginate method should be already called.
     *
     * @see paginate()
     * @return PaginatorInterface
     */
    public function getPaginator();
}
```

If you wish to simlify pagination support in your objects you can also use `PaginatorTrait` which will implement following methods and provide way to use default spiral paginator linked to `ServerRequestInterface`. Let's try to provide an example of paginating object using such trait:

```php
public function index()
{
    $selection = $this->dbal->db()->users->select('name','email');
    dump($selection->paginate(1)->getIterator());
}
```

Default spiral paginator `Paginator` will automatically fetch count of records from paginable object and set approparite limit/offset values based on provided parameters. Page pagination will be performed based on query parameter `page`. You can always ajust parameter or [**request**] (/http/flow.md) paginator must follow:

```php
public function index()
{
    $selection = $this->dbal->db()->users->select('name','email');
    dump($selection->paginate(1, 'myPage', $this->request)->getIterator());
}
```

If you wish to change Uri paginator must follow (to generate page Uris) you use method `setUri()` (attention, such method is available only in default spiral Paginator not in it's interface):

```php
public function index()
{
    $selection = $this->dbal->db()->users->select('name','email')->paginate(10);
    dump($selection->getIterator());

    $selection->getPaginator()->setUri(new Uri('/custom-uri/'));
    dump($selection->getPaginator()->createUri(1));
}
```

## PaginatorInterface
If you wish to create your own paginator, for example to fetch page number from Route parameter you can easily do that by implementing `PaginatorInterface`:

```php
interface PaginatorInterface
{
    /**
     * Set page number.
     *
     * @param int $number
     * @return int
     */
    public function setPage($number);

    /**
     * Get current page number.
     *
     * @return int
     */
    public function getPage();

    /**
     * Apply pagination to a simple array, should fetch count from target array and return sliced
     * array version.
     *
     * @param array $haystack Target array must be paginated.
     * @return array
     * @throws PaginationException
     */
    public function paginateArray(array $haystack);

    /**
     * Apply paginator to paginable object.
     *
     * @param PaginableInterface $object
     * @return PaginableInterface
     * @throws PaginationException
     */
    public function paginateObject(PaginableInterface $object);

    /**
     * Create page URL using specific page number. No domain or schema information included by
     * default, starts with path.
     *
     * @param int $pageNumber
     * @return UriInterface
     */
    public function createUri($pageNumber);
}
```

For example, let's pretend that we have an implemention which wants as instance of Route and parameters used to generate url.

```php
public function index($page = 1)
{
    $selection = $this->dbal->db()->users->select('name','email');
    $selection->setPaginator(
        new MyPaginator(
            $this->router->activeRoute(),               //We can use active route
            'page',                                     //Parameter for page
            $page,                                      //Parameter value
            ['controller'=>'home', 'action'=>'index']    //Route options
        )
    );

    dump($selection->getIterator());
}
```

## Render paginator on page
Spiral does not include rendering logic into pagination class itself, you have to render page range by youself. Luckily, if you have installed module `spiral/toolkit` you can use [virtual templater tag/widget] (/templater/expert.md) for such purposes. Such widget located in 'spiral:paginator' tag and can accept both `PaginableInterface` and `PaginatorInterface` as source however it can work only with default spiral `Paginator` class.

```php
public function index()
{
    $selection = $this->dbal->db()->users->select('name','email')->paginate(2);
    return $this->views->render('users/list', ['list' => $selection]);
}
```

View source:

```html
<extends:spiral:layouts.blank page="Users list"/>

<block:content>
    <table>
        <thead>
        <tr>
            <th>[[Name:]]</th>
            <th>[[Email:]]</th>
        </tr>
        </thead>
        <tbody>
        <?php
        /**
         * @var array[]|\Spiral\Pagination\PaginableInterface
         */
        foreach ($list as $user) {
            ?>
            <tr>
                <td><?= e($user['name']) ?></td>
                <td><?= e($user['email']) ?></td>
            </tr>
            <?php
        }
        ?>
        </tbody>
    </table>
    
    <spiral:paginator source="<?= $list ?>"/>
</block:content>
```

Given example will produce output like that:
![Pagination](https://raw.githubusercontent.com/spiral/guide/master/resources/pagination.png)

You can create your own project/module paginators using Templater virtual tags (widgets).

## JSON packing
There is few scenarious when you would like to include pagination data into resulted JSON response, and again, in order to simplify default implementations, spiral does not provide such ability out of the box, such code will return only array of paginated data (select builder implements `JsonSerializable`):

```php
public function index()
{
    $selection = $this->dbal->db()->users->select('name', 'email')->paginate(2);

    return $selection;
}
```

However we can ealisy implement our own JSON packer with desired pagination format, let's create class for that:

```php
class JSONPaginator implements JsonSerializable
{
    protected $select = null;

    public function __construct(PaginableInterface $select)
    {
        if (!$select->isPaginated()) {
            throw new RuntimeException('Selection must be paginated.');
        }
        
        $this->select = $select;
    }

    public function jsonSerialize()
    {
        $paginator = $this->select->getPaginator();

        return [
            'result'     => $this->select,
            'pagination' => [
                'count'        => $paginator->count(),
                'page'         => $paginator->getPage(),
                'nextPage'     => $paginator->nextPage(),
                'previousPage' => $paginator->previousPage()
            ]
        ];
    }
}
```

Now we can modify our controller code to use our pagination wrapper:

```php
public function index()
{
    $selection = $this->dbal->db()->users->select('name', 'email')->paginate(2);

    return new \JSONPaginator($selection);
}
```

And our result will look like:

```json
{
    "result": [
        {
            "name": "Anton",
            "email": "anton@email.com"
        },
        {
            "name": "John",
            "email": "john@email.com"
        }
    ],
    "pagination": {
        "count": 5,
        "page": 1,
        "nextPage": 2,
        "previousPage": false
    }
}
```
