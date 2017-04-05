## Selections
Once your collection is populated with data, it's time to select some of the Users to perform something. The Spiral ODM exposes 3 ActiveRecord like methods which you can use for these purposes: `find`, `findOne` and `findByPK`.

There is 4th method which is used inside every find method - `selector`. You can use it to get access to the  `DocumentSelector` class associated with our document. You can also get the associated collecion by calling the method `selector` of the ODM component.

#### FindOne
To find a document based on some conditions and sorting, we can use the static method `findOne`. This method will return null if it's unabled to locate the required document. 
The ODM component does not provide any query wrapper at the top of the MongoDB so you are able to use the pure mongo queries for your requests:

```php
//Don't worry about second argument yet, it's about pre-loading
$user = User::findOne(['email' => 'test@email.com'];
dump($user);
```

> The ODM provides only one feature of automatic conversion of the `DateTime` classes in your queries into instance of `MongoDate`.

#### FindByPK
When you want to find your document based on it's primary key value (MongoId in "_id" field), you can use the method `findByPK`. This method will behave the same way as `findOne` if no entity is found.

```php
$user = User::findByPK($mongoID);
dump($user);
```

> Method can accept both `MongoId` and string which will be automatically combined to valid id instance.

The most common option is to find the model based on the provided primary key or return client exception on failure. Spiral does not embed any application specific logic into the ODM component so you have to raise an exception on your own. This is easy to do:

```php
protected function indexAction($id)
{
    if (empty($user = User::findByPK((int)$id))) {
        throw new NotFoundException('No such user');
    }

    dump($user);
}
```

Or using source:

```php
protected function indexAction($id, DocumentSource $users)
{
    if (empty($user = $users->findByPK($id))) {
        throw new NotFoundException('No such user');
    }

    dump($user);
}
```

#### Find and Working with Selector
To find one or multiple document instances from desired collection, we have to first get the associated instance of `Collecion`. Then we can do this using the ODM component method `selector` or via the static function `find` of our User model.

```php
$selector = User::find();
```

> You also have an way to connect to your source with set of pre-defined methods using static function `source` of your model.

```php
foreach(Users::source()->findActive() as $user) {
    //...
}
```

You are able to set or clarify the selection query using the methods `query` or `where`:

```php
$collection = User::find()->where([
    'email' => new \MongoRegex('/test@email.com/')
]);

dump($collection->count());
```

In addition, you are able to paginate your result using the techniques described in [Pagination](/components/pagination.md) component:

```php
$collection = User::find()->paginate(10);
```

> Check other Collection methods, such as limit, offset (skip) and sortBy.

##### Fetching Documents
Once you've configured your Collection selection with desired query, you can fetch the found document by simply iterating over collection or explicitly calling the method `getIterator`/`getCursor`, which will return an instance of `DocumentCursor`:

```php
foreach (User::find()->paginate(10) as $user) {
    dump($user);
}
```

```php
$cursor = User::find()->sortBy(['name' => 1])->paginate(10)->getCursor();

/**
 * @var DocumentCursor $cursor
 */
foreach ($cursor as $user) {
    dump($user);
}
```

##### DocumentCursor
The provided instance of `DocumentCursor` wraps the top of the `MongoCursor` and provides support for lazy creation of the collection documents (no entity cache is used). As a result, you can iterate over a huge amount of Documents without any significant memory usage. Also you can access the `MongoCursor` functionality:

```php
$cursor = User::find()->sortBy(['name' => 1])->paginate(10)->getCursor();

/**
 * @var DocumentCursor $cursor
 * @var User           $user
 */
while ($user = $cursor->getNext()) {
    dump($user);
    dump($cursor->dead());
}

dump($cursor->info());
```

##### Fetching Fields
If you wish to fetch only the specified set of fields from your collection, simply specify their names in the form of an array into the `fetchFields` method:

```php
foreach (User::find()->fetchFields(['name']) as $data) {
    dump($data);
}
```

> Attention, fetchFields() will return one big array for whole selection, do not use it for big datasets. Use alternative curson configuration:

```php
foreach (User::find()->getCursor()->fields(['name' => true]) as $data) {
    dump($data);
}
```

This methods will read all the desired fields and return them in the form of an array. You can also read them in a streaming mode by utilizing the `fields` method of your `DocumentCursor`:

```php
$cursor = User::find()->getCursor()->fields([
    'email' => true
]);

while ($data = $cursor->getNext()) {
    dumP($data);
}

dumP($cursor->info());
```

> Attention, format in this case follows the [offical documentation] (http://php.net/manual/ru/mongocursor.fields.php)!
