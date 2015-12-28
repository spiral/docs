# Database and Data Entities
One of the critical part of any application is set of models dedicated to represent data stored in some permanent location. At this moment spiral provides two generic data entities you can use in your application - `ORM\Record` and `ODM\Document`. Both entities can be scaffolded using console toolkit and will be located in 'app/classes/Database' directory.

```php
class Sample extends Record
{
    //time_created, time_updated
    use TimestampsTrait;

    /**
     * @var string
     */
    protected $database = 'sandbox';

    /**
     * @var array
     */
    protected $schema = [
        'id'      => 'primary',
        'status'  => 'enum(active,disabled)',
        'name'    => 'string',
        'content' => 'text',
        
        'child'   => [
            self::HAS_ONE => SampleChild::class,
            self::INVERSE => 'sample'
        ],
    ];
}
```

[ORM](/orm/basics.md) and [ODM](/odm/basics.md) entities provide `ActiveRecord` like behaviour so you can use them directly in your code, hovewer it's recommended to look at [Models and Services](/application/services.md) for this purposes as it will allow you to make your code more modular and isolate database specific operations from your controllers.

> You can also check Source models for ORM and ODM entities which can provide you ability to pre-build your queries and isolate database operations from service layer of your application.

```php
class SampleSource extends RecordSource implements SingletonInterface
{
    const SINGLETON = self::class;

    /**
     * Linked record
     */
    const RECORD = Sample::class;

    /**
     * @return \Spiral\ORM\Entities\RecordSelector
     */
    public function findAll()
    {
        return $this->find()->load('child');
    }

    /**
     * @return \Spiral\ORM\Entities\RecordSelector
     */
    public function findActive()
    {
        return $this->find(['status' => 'active'])->load('child');
    }

    /**
     * Active records with specific value in child table.
     *
     * @param int $value
     * @return \Spiral\ORM\Entities\RecordSelector
     */
    public function findByValue($value)
    {
        //Alternative: ->where('child.value', '=', new Parameter($value, \PDO::PARAM_INT))
        return $this->findAll()->with('child')->where('child.value', '=', (int)$value);
    }

    /**
     * @param Sample $entity
     * @param array  $errors Reference
     * @return bool
     */
    public function save(Sample $entity, &$errors = null)
    {
        if (!$entity->save()) {
            $errors = $entity->getErrors();
            return false;
        }

        return true;
    }
}
```

Spiral can link your sources to needed entity automatically and allow you to use them via DI.

```php
public function indexAction(SampleSource $source)
{
   echo $source->findActive()->count();
}
```

## Databases
Spiral Framework provides set of tools used to split, separate or join together multiple database sources (physically or logically), on practice it means that you can write your code relying on multiple databases even if internally they all linked together. Such approach can give your code ability to be moved to other spiral (or not) application with only few config corrections.

For example following databases configuration will create two virtual databases which are both located in one place but are separated using unique table prefix:

```php
'primary'   => [
    'connection'  => 'mysql',
    'tablePrefix' => 'primary_'
],
'secondary' => [
    'connection'  => 'mysql',
    'tablePrefix' => 'secondary_',
],
```

> Separatelly from that database isolation and database aliases can be used to write more powerful extensions or even application inside application.

## What is [DataEntity](/components/entity.md)
Generic purposes of any entity is to provide access to its data using set of getters, setters and accessors. In addition to that spiral count that every entity model can and must be validated before any storage related operations. You can read more about base DataEntity [here](/components/entity.md).
