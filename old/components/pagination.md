# Pagination
Spiral database, ORM and ODM layers support ability to specify limit and offset values based on external pagination object.

## PaginatorInterface
Pagination interface is defined using following contract:

```php
interface PaginatorInterface
{
    /**
     * Get pagination limit value.
     *
     * @return int
     */
    public function getLimit(): int;

    /**
     * Get calculated offset value.
     *
     * @return int
     */
    public function getOffset(): int;

    /**
     * Get parameter paginator depends on. Environment specific.
     *
     * @return null|string
     */
    public function getParameter();
}
```

Classes which are open for pagination must implement `PaginatorAwareInterface`:

```php
interface PaginatorAwareInterface
{
    /**
     * Indication that object has associated paginator.
     *
     * @return bool
     */
    public function hasPaginator(): bool;

    /**
     * Manually set paginator instance for specific object.
     *
     * @param PaginatorInterface $paginator
     */
    public function setPaginator(PaginatorInterface $paginator);

    /**
     * Get paginator for the current selection. Paginate method should be already called or
     * paginator must be previously set.
     *
     * Potentially to be renamed to getPaginator method since this method does not create paginator
     * automatically.
     *
     * @see paginate()
     *
     * @return PaginatorInterface
     *
     * @throws PaginationException
     */
    public function getPaginator(): PaginatorInterface;
}
```

Default pagination implementation can be received using `Spiral\Pagination\Paginator` class, we can use it directly in our code:

```php
$paginator = new Paginator(10);

$selection = $this->db->users->select('name', 'email');
$selection->setPaginator($paginator->withPage(2));
```

### Http based pagination
In most of cases you want to associate paginator with specific query page parameter, you can either do
it manually or use `PaginatorsInterface` which is based on active request scope:

```php
$paginator = $this->paginators->createPaginator('query-parameter', 10);

$selection = $this->db->users->select('name', 'email')->paginate();
$selection->setPaginator($paginator);
```

In addition, DBAL, ORM and ODM components include shortcut method which will initiate paginator automatically:

```php
//Default page parameter is "page"
$selection = $this->db->users->select('name', 'email')->paginate(10, 'query-parameter');
```


## Render paginator on page
Spiral does not include rendering logic into pagination class itself, you have to render page range by youself.
 
Luckily, if you have installed `spiral/toolkit` you can use [widget](/old/stempler/expert.md) "<spiral:paginator>" for such purposes. Widget can accept both `PaginatorInterface` and `PaginatorAwareInterface`:

```php
protected function indexAction()
{   
    return $this->views->render('users/list', [
        'list' => $this->db->users->select('name','email')->paginate(2)
    ]);
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

Result:
![Pagination](https://raw.githubusercontent.com/spiral/guide/master/resources/pagination.png)

> You can create your own project/module paginators using Stempler virtual tags (widgets).

## JSON packing
You can create your own pagination responses by talking to `Paginator` class directly:

```php
class ResponseWrapper implements JsonSerializable
{
    protected $select;

    public function __construct(PaginatorAwareInterface $select)
    {
        if (!$select->hasPaginator()) {
            throw new RuntimeException('Selection must be paginated');
        }
        
        if ($select->getPaginator() instanceof PagedIterface) {
            throw new RuntimeException('PagedIterface compatible paginator is required');
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

```php
protected function indexAction()
{
    $selection = $this->dbal->db()->users->select('name', 'email')->paginate(2);

    return new \ResponseWrapper($selection);
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
