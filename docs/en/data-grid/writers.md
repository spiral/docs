# Custom Writers

Writers are responsible for translating Data Grid specifications into actual data operations. They convert abstract
filter, sorter, and pagination specifications into concrete queries, API calls, or collection operations for your
specific data source.

## Basic Setup

The Data Grid compiler uses writers to transform specifications into executable operations. Writers are registered in
your configuration and are applied in order to find the first writer that can handle each specification.

### Spiral Framework Configuration

```php
// app/config/dataGrid.php
return [
    'writers' => [
        \Spiral\Cycle\DataGrid\Writer\QueryWriter::class,
        \Spiral\Cycle\DataGrid\Writer\PostgresQueryWriter::class,
        \Spiral\Cycle\DataGrid\Writer\BetweenWriter::class,
        \App\DataGrid\Writer\CustomCollectionWriter::class, // Your custom writer
    ],
];
```

### Manual Compiler Configuration

When using the Data Grid component outside of the Spiral Framework, you need to manually configure the compiler with
your custom writers. This gives you complete control over how specifications are processed while maintaining the
flexibility of the component.

```php
use Spiral\DataGrid\Compiler;
use Spiral\DataGrid\GridFactory;
use Spiral\DataGrid\GridFactoryInterface;
use Spiral\DataGrid\Input\ArrayInput;
use Spiral\DataGrid\Grid;

// Create and configure the compiler
$compiler = new Compiler();

// Add your custom writers in order of priority
$compiler->addWriter(new ElasticsearchWriter());
$compiler->addWriter(new DoctrineCollectionWriter());
$compiler->addWriter(new ArrayCollectionWriter());

// Configure the grid factory
$gridFactory = new GridFactory(
    compiler: $compiler,
    input: new ArrayInput($_GET), // or your preferred input source
    view: new Grid(),
);

// Create schema
$schema = new GridSchema();
$schema->addFilter('name', new Filter\Like('name', new StringValue()));
$schema->addSorter('name', new Sorter\DirectionalSorter('name'));
$schema->setPaginator(new PagePaginator(10, [10, 25, 50]));

// Use with different data sources
$data = [
    ['name' => 'John', 'age' => 30],
    ['name' => 'Jane', 'age' => 25],
    ['name' => 'Bob', 'age' => 35],
];

$grid = $gridFactory->create($data, $schema);
$result = $grid->getItems(); // Filtered, sorted, and paginated data
```

> **Note**: The order matters - writers are checked sequentially until one accepts the specification.

## Creating Custom Writers

### Basic Writer Interface

### Doctrine DBAL Writer

```php
use Doctrine\DBAL\Connection;
use Doctrine\DBAL\Query\QueryBuilder;

class DoctrineDBALWriter implements WriterInterface
{
    public function __construct(private readonly Connection $connection) {}
    
    public function write(mixed $source, SpecificationInterface $specification, Compiler $compiler): mixed
    {
        if (!$source instanceof QueryBuilder) {
            return null;
        }
        
        return match (true) {
            $specification instanceof Filter\Equals => $this->writeEquals($source, $specification),
            $specification instanceof Filter\Like => $this->writeLike($source, $specification),
            $specification instanceof Filter\Gt => $this->writeGt($source, $specification),
            $specification instanceof Filter\Lt => $this->writeLt($source, $specification),
            $specification instanceof Sorter\AscSorter => $this->writeSorter($source, $specification, 'ASC'),
            $specification instanceof Sorter\DescSorter => $this->writeSorter($source, $specification, 'DESC'),
            $specification instanceof Pagination\Limit => $source->setMaxResults($specification->getValue()),
            $specification instanceof Pagination\Offset => $source->setFirstResult($specification->getValue()),
            default => null
        };
    }
    
    private function writeEquals(QueryBuilder $qb, Filter\Equals $filter): QueryBuilder
    {
        return $qb->andWhere($qb->expr()->eq($filter->getExpression(), ':value'))
                  ->setParameter('value', $filter->getValue());
    }
    
    private function writeLike(QueryBuilder $qb, Filter\Like $filter): QueryBuilder
    {
        return $qb->andWhere($qb->expr()->like($filter->getExpression(), ':value'))
                  ->setParameter('value', '%' . $filter->getValue() . '%');
    }
    
    private function writeGt(QueryBuilder $qb, Filter\Gt $filter): QueryBuilder
    {
        return $qb->andWhere($qb->expr()->gt($filter->getExpression(), ':value'))
                  ->setParameter('value', $filter->getValue());
    }
    
    private function writeLt(QueryBuilder $qb, Filter\Lt $filter): QueryBuilder
    {
        return $qb->andWhere($qb->expr()->lt($filter->getExpression(), ':value'))
                  ->setParameter('value', $filter->getValue());
    }
    
    private function writeSorter(QueryBuilder $qb, Sorter\AscSorter|Sorter\DescSorter $sorter, string $direction): QueryBuilder
    {
        foreach ($sorter->getExpressions() as $expression) {
            $qb = $qb->addOrderBy($expression, $direction);
        }
        
        return $qb;
    }
}
```

### Collection Writer

Here's an example of a writer that works with PHP collections:

```php
use Spiral\DataGrid\Compiler;
use Spiral\DataGrid\SpecificationInterface;
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Sorter;
use Spiral\DataGrid\Specification\Pagination;
use Spiral\DataGrid\WriterInterface;
use Doctrine\Common\Collections\Collection;
use Doctrine\Common\Collections\Criteria;

class DoctrineCollectionWriter implements WriterInterface
{
    public function write(mixed $source, SpecificationInterface $specification, Compiler $compiler): mixed
    {
        if(\is_array($source)) {
            $source = new ArrayCollection($source);
        }
        
        // Only accept if source is a Doctrine Collection
        if (!$source instanceof Collection) {
            return null;
        }
        
        $criteria = Criteria::create();
        
        // Handle different specification types
        match (true) {
            $specification instanceof Filter\Equals => $this->writeEquals($specification, $criteria),
            $specification instanceof Filter\Like => $this->writeLike($specification, $criteria),
            $specification instanceof Filter\Gt => $this->writeGreaterThan($specification, $criteria),
            $specification instanceof Filter\Lt => $this->writeLessThan($specification, $criteria),
            $specification instanceof Sorter\AscSorter => $this->writeAscSorter($specification, $criteria),
            $specification instanceof Sorter\DescSorter => $this->writeDescSorter($specification, $criteria),
            $specification instanceof Pagination\Limit => $this->writeLimit($specification, $criteria),
            $specification instanceof Pagination\Offset => $this->writeOffset($specification, $criteria),
            default => null
        };
        
        return $source->matching($criteria);
    }
    
    private function writeEquals(Filter\Equals $filter, Criteria $criteria): void
    {
        $criteria->andWhere(Criteria::expr()->eq($filter->getExpression(), $filter->getValue()));
    }
    
    private function writeLike(Filter\Like $filter, Criteria $criteria): void
    {
        $criteria->andWhere(Criteria::expr()->contains($filter->getExpression(), $filter->getValue()));
    }
    
    private function writeGreaterThan(Filter\Gt $filter, Criteria $criteria): void
    {
        $criteria->andWhere(Criteria::expr()->gt($filter->getExpression(), $filter->getValue()));
    }
    
    private function writeLessThan(Filter\Lt $filter, Criteria $criteria): void
    {
        $criteria->andWhere(Criteria::expr()->lt($filter->getExpression(), $filter->getValue()));
    }
    
    private function writeAscSorter(Sorter\AscSorter $sorter, Criteria $criteria): void
    {
        foreach ($sorter->getExpressions() as $expression) {
            $criteria->orderBy([$expression => Criteria::ASC]);
        }
    }
    
    private function writeDescSorter(Sorter\DescSorter $sorter, Criteria $criteria): void
    {
        foreach ($sorter->getExpressions() as $expression) {
            $criteria->orderBy([$expression => Criteria::DESC]);
        }
    }
    
    private function writeLimit(Pagination\Limit $limit, Criteria $criteria): void
    {
        $criteria->setMaxResults($limit->getValue());
    }
    
    private function writeOffset(Pagination\Offset $offset, Criteria $criteria): void
    {
        $criteria->setFirstResult($offset->getValue());
    }
}
```

### API Client Writer

For external API data sources:

```php
use GuzzleHttp\Client;
use Spiral\DataGrid\Compiler;
use Spiral\DataGrid\SpecificationInterface;
use Spiral\DataGrid\WriterInterface;

class ApiClientWriter implements WriterInterface
{
    public function __construct(
        private readonly Client $httpClient,
        private array $queryParams = []
    ) {}
    
    public function write(mixed $source, SpecificationInterface $specification, Compiler $compiler): mixed
    {
        // Accept if source is our API client
        if (!$source instanceof ExternalApiClient) {
            return null;
        }
        
        // Convert specifications to API query parameters
        match (true) {
            $specification instanceof Filter\Equals => $this->addFilter($specification),
            $specification instanceof Filter\Like => $this->addSearch($specification), 
            $specification instanceof Sorter\AscSorter => $this->addSort($specification, 'asc'),
            $specification instanceof Sorter\DescSorter => $this->addSort($specification, 'desc'),
            $specification instanceof Pagination\Limit => $this->addLimit($specification),
            $specification instanceof Pagination\Offset => $this->addOffset($specification),
            default => null
        };
        
        // Return modified source with accumulated parameters
        return $source->withQueryParams($this->queryParams);
    }
    
    private function addFilter(Filter\Equals $filter): void
    {
        $this->queryParams['filter'][$filter->getExpression()] = $filter->getValue();
    }
    
    private function addSearch(Filter\Like $filter): void
    {
        $this->queryParams['search'][$filter->getExpression()] = $filter->getValue();
    }
    
    private function addSort(Sorter\AscSorter|Sorter\DescSorter $sorter, string $direction): void
    {
        foreach ($sorter->getExpressions() as $expression) {
            $this->queryParams['sort'][$expression] = $direction;
        }
    }
    
    private function addLimit(Pagination\Limit $limit): void
    {
        $this->queryParams['limit'] = $limit->getValue();
    }
    
    private function addOffset(Pagination\Offset $offset): void
    {
        $this->queryParams['offset'] = $offset->getValue();
    }
}
```

### Elasticsearch Writer

For Elasticsearch queries:

```php
use Elasticsearch\Client;
use Spiral\DataGrid\Compiler;
use Spiral\DataGrid\SpecificationInterface;
use Spiral\DataGrid\WriterInterface;

class ElasticsearchWriter implements WriterInterface
{
    public function __construct(
        private readonly Client $client,
        private array $query = ['bool' => ['must' => []]],
    ) {}
    
    public function write(mixed $source, SpecificationInterface $specification, Compiler $compiler): mixed
    {
        if (!$source instanceof ElasticsearchQueryBuilder) {
            return null;
        }
        
        match (true) {
            $specification instanceof Filter\Equals => $this->addTermQuery($specification),
            $specification instanceof Filter\Like => $this->addMatchQuery($specification),
            $specification instanceof Filter\Between => $this->addRangeQuery($specification),
            $specification instanceof Sorter\AscSorter => $this->addSort($specification, 'asc'),
            $specification instanceof Sorter\DescSorter => $this->addSort($specification, 'desc'),
            $specification instanceof Pagination\Limit => $this->addSize($specification),
            $specification instanceof Pagination\Offset => $this->addFrom($specification),
            default => null
        };
        
        return $source->withQuery($this->query);
    }
    
    private function addTermQuery(Filter\Equals $filter): void
    {
        $this->query['bool']['must'][] = [
            'term' => [
                $filter->getExpression() => $filter->getValue(),
            ],
        ];
    }
    
    private function addMatchQuery(Filter\Like $filter): void
    {
        $this->query['bool']['must'][] = [
            'match' => [
                $filter->getExpression() => [
                    'query' => $filter->getValue(),
                    'fuzziness' => 'AUTO',
                ],
            ],
        ];
    }
    
    private function addRangeQuery(Filter\Between $filter): void
    {
        [$min, $max] = $filter->getValue();
        $this->query['bool']['must'][] = [
            'range' => [
                $filter->getExpression() => [
                    'gte' => $min,
                    'lte' => $max,
                ],
            ],
        ];
    }
    
    private function addSort(Sorter\AscSorter|Sorter\DescSorter $sorter, string $direction): void
    {
        foreach ($sorter->getExpressions() as $expression) {
            $this->query['sort'][] = [
                $expression => ['order' => $direction],
            ];
        }
    }
    
    private function addSize(Pagination\Limit $limit): void
    {
        $this->query['size'] = $limit->getValue();
    }
    
    private function addFrom(Pagination\Offset $offset): void
    {
        $this->query['from'] = $offset->getValue();
    }
}
```