# Document and DocumentEntity
ODM models are represent by two primary models `Document` and `DocumentEntity`.
In opposite to ORM `DocumentEntity` can not be saved to database directly and intended to used as embedded into parent `Document` as singular property or array of documents.

> Examples are given using `SourcesTrait` for simplicity.

## Document
The ODM classes Document (and `DocumentEntity`, see in extended usage) only lets you manipulate a set of fields described in the model schema. The declaration of fields can be performed by simply creating an associated array between the field name and it's **type**. 

```php
class User extends Document
{
    const SCHEMA = [
        'id'      => ObjectId::class,
        'name'    => 'string',
        'email'   => 'string',
        'balance' => 'float'
    ];
}
```

```php
$user = new User();
$user->name = 'Antony';
$user->save(); //All documents are AR
```

### Default Values
ODM will automatically force the default values based on the typecasted null value (string => "", int => 0). You can set your own default values using the model constant "DEFAULTS":

```php
const DEFAULTS = [
    'balance' => 100.00    
];
```

#### Collection and Database
By default, Spiral will associate your `Document` model with the default ODM database and collection based on class name (in our case User => users). You can alter both of these values by declaring non empty constants like `DATABASE` and `COLLECTION`.

#### Indexes
To declare which indexes are created in the associated collection, simply declare them in the `INDEXES` constant of your Document. The Declaration form is similar to the MongoCollection ensureIndexes method. Indexes will be created at the moment of schema synchronization (see next).

```php
const INDEXES = [
    ['email' => 1, '@options' => ['unique' => true]],
    ['name' => 1, 'balance' => -1]
];
```

> Use "@options" key to specify the index options.

In order to force ODM component to create collection indexes run command `odm:schema` with `-i` flag.

## DocumentEntity
Declaration of DocumentEntity is identical to Document, though you are not able to associate database and collection with it.

```php
class Profile extends DocumentEntity
{
    const SCHEMA = [
        'bio' => 'string'
    ];
}
```

## Embed One
To embed DocumentEntity to your model use it's class name as property type:

```php
const SCHEMA = [
    'profile' => Profile::class
];
```

```php
$user->profile->bio = 'new bio';
```

> Make sure to set `SECURED` or `FILLABLE` constant in your parent model to allow filling embedded model values using `setFields` method:

```php
$user->setFields([
    'profile' => [
        'bio' => 'new value'
    ]
]);
```

## Embed Multiple
To embed array of sub-models declare type as array:

```php
const SCHEMA = [
    'profiles' => [Profile::class]
];
```

```php
$user->profiles->add(new Profile(...));
```

#### Modifying Documents schema
Since ODM schema is not stored in the database, it can be efficiently updated without any storage change. As a result you are able to add new columns, compositions etc into your Documents at any moment. Simply declare them in your model schema and update the ODM after.

```php
const SCHEMA = [
    'id'      => ObjectId:class,
    'name'    => 'string',
    'email'   => 'string',
    'balance' => 'float',
    'data'    => 'string'
];
```

## Create Document
Once your ODM schema has been updated, we are ready to work with our models. To save some data into our users collection let's create our first entity in our controller method:

```php
protected function indexAction()
{
    $u = new User();
    $u->name = 'Anton';
    $u->email = 'test@email.com';
    $u->balance = 99;
    $u->save();

    dump($u);
}
```

Alternatively, if you using DocumentSource you can create your document this way:

```php
protected function indexAction(DocumentSource $source)
{
    $u = $source->create($this->input->data);
    $u->save();

    dump($u);
}
`````

## Modify Document
Once you've selected the necessary document (for example using `findByPK`), you can modify it's fields the same way as any other `DataEntity`. To save your updates into the database call method `save()`.

```php
protected function indexAction($id)
{
    $user = User::findByPK($id);
    $user->name = 'New Name';

    $user->save();
}
```

#### Dirty Fields and Solid State
If you dump the User models before saving them into the database, you will see the property "atomics" which demonstrates what query will be sent to the MongoDB to perform the requested updated. 

```
atomics:dynamic = array(1)
(
·    ['$set'] = array(1)
·    (
·    ·    ['name'] = string(8) New Name
·    )
)
```

As you might notice, Document sends only updated values for the name field using the atomic operation set. This approach is useful in many cases when the data is updated from many places. However if you want to save your entity data into the database as a solid dataset, you can force the entity "solid state". When the model (Document) is in a solid state, every small change will create a full dataset update.

```php
$user = User::findByPK($id);
$user->name = 'New Name';

$user->solidState(true);
dump($user);
```

Now our atomic operation will look like this:

```
atomics:dynamic = array(1)
(
·    ['$set'] = array(4)
·    (
·    ·    ['balance'] = double(5) 27.920000000000002
·    ·    ['name'] = string(8) New Name
·    ·    ['email'] = string(22) klangworth@hotmail.com
·    )
)
```

> If you want to keep our model in solid state by default, you can call this method in the overwritten model constructor. SolidState automatically applies to every model that is created.


## Delete Documents
You can also delete any existing Document by executing it's method "delete". 

## Inheritance and Abstract Documents
Since ODM uses static analysis, there are no real limitations on how you can create your models. As a result you can declare the **abstract** Document and later extend this class in your application models. This technique can be very useful when writing [modules](/framework/modules.md).

> When extending, ODM will merge schemas and other properties of the Document and it's parent.

**You can also use the data driven model inheritance, check [this article](/odm/oop.md).**