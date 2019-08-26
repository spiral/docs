# Scaffolding
You can scaffold ODM class Document and pre-define it's fields using `spiral create:document` command.

> Note, there is no DocumentEntity command, simple change parent class.

`spiral create:document user -f _id:MongoId -f name:string`

```php
class User extends Document
{
    const SCHEMA = [
        '_id'  => MongoId::class,
        'name' => 'string'
    ];
}
```

> Use command option -s to automatically generate repository class.

As in case with ORM you have to update your schema to register class: `spiral odm:schema`.

> Run `spiral odm:schema -i` to automatically add all requested mongo indexes.
> Run `spiral ide-helper` to generate IDE tooltips.
