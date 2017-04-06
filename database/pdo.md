# PDO Queries
DBAL component is based on PDO extension. You have full access to PDO functionality including ability to write custom SQL queries and work with PDOStatements.

## Query Database
You can query your database using prepared PDO statement following way:

```php
$statement = $this->db->query('SELECT COUNT(*) FROM users WHERE id > ?', [1]);

assert($statement instanceof \PDOStatement);
dump($statement);
```

Named parameters are also supported:

```php
$statement = $this->db->query('SELECT COUNT(*) FROM users WHERE id > :id', [
    ':id' => 2
]);

assert($statement instanceof \PDOStatement);
dump($statement);
```

> Use [Query Builders](/database/builders.md) to let DBAL compile DBMS specific query for you.

## Access PDO
If you wish to access PDO instance associated with database use:

```php
dump($this->db->getDriver()->getPDO());
```