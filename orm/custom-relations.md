# Custom Relations
Spiral ORM engine can be extend in order to support custom relation mechanisms including: schema generation, eager-loading, post-loading, data representation and ect.

> Take a look at [ODM Bridge](/orm/odm-bridge.md) to check an example of custom relations.

## Implementations
Each custom relation must be presented by set of 3 implementations:

Interface                    | Description
---                          | ---
Spiral\ORM\RelationInterface | Used to represent loaded data in parent entity.
Spiral\ORM\Schemas\RelationInterface | Used to compile ORM schema, declared needed columns and FKs.
Spiral\ORM\LoaderInterface | Configures RecordSelector based on relation options and JOIN needed tables

Once your relation is implemented you can register in in schemas/relations config based on desired name:

```php
'customRelation'  => [
    RelationsConfig::SCHEMA_CLASS => Vendor\CustomRelation\CustomRelationSchema::class,
    RelationsConfig::LOADER_CLASS => Vendor\CustomRelation\CustomRelationLoader::class,
    RelationsConfig::ACCESS_CLASS => Vendor\CustomRelation\CustomRelation::class
],
```

Use relation in your models same way as any other relation:

```php
const SCHEMA = [
    'id' => 'bigPrimary', 
    'my-relation' => [
        'customRelation' => Model::class,
        //options
    ]
];
```

> You are to link you relations to any desired model, even outside of ORM.

## Eager Loading
ORM engine converts your related data into tree using combination of `LoaderInterface` and set of tree Node classes.

> Note that all loaders are initiated using ORM specific container (usually global application container), this makes you able to request external dependencies needed to load your data.

We can demonstrate how to collect FK references and load data using `HasDocumentLoader`: 

```php
class HasDocumentLoader extends Component implements LoaderInterface
{
    /**
     * @var string
     */
    protected $class;

    /**
     * @invisible
     * @var ODMInterface
     */
    protected $odm;

    /**
     * Parent loader if any.
     *
     * @invisible
     * @var LoaderInterface
     */
    protected $parent;

    /**
     * Loader options, can be altered on RecordSelector level.
     *
     * @var array
     */
    protected $options = [];

    /**
     * Relation schema.
     *
     * @var array
     */
    protected $schema = [];

    /**
     * @param string       $class
     * @param array        $schema Relation schema.
     * @param ODMInterface $odm
     */
    public function __construct(string $class, array $schema, ODMInterface $odm)
    {
        $this->class = $class;
        $this->schema = $schema;
        $this->odm = $odm;
    }

    /**
     * @return string
     */
    public function getClass(): string
    {
        return $this->class;
    }

    /**
     * {@inheritdoc}
     */
    public function withContext(LoaderInterface $parent, array $options = []): LoaderInterface
    {
        if (!empty($options)) {
            throw new LoaderException("HasDocumentLoader does not support any options");
        }

        $loader = clone $this;
        $loader->parent = $parent;
        $loader->options = $options + $this->options;

        return $loader;
    }

    /**
     * {@inheritdoc}
     */
    public function createNode(): AbstractNode
    {
        return new SingularNode(
            [/* since columns are empty all document properties will be used */],
            $this->schema[Record::OUTER_KEY],
            $this->schema[Record::INNER_KEY],
            '_id'
        );
    }

    /**
     * {@inheritdoc}
     */
    public function loadData(AbstractNode $node)
    {
        if (empty($node->getReferences())) {
            return;
        }

        //Loading document data
        $cursor = $this->odm->selector($this->class)->where([
            $this->schema[Record::OUTER_KEY] => ['$in' => $node->getReferences()]
        ])->getProjection();

        //Pushing documents data into data tree
        foreach ($cursor->toArray() as $document) {
            $node->parseRow(0, $document);
        }
    }
}
```