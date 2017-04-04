# Scaffolding and Migrations
Spiral ORM ships with embedded mechanism to analyse requested database structure and generate set of needed migrations to alter db structure.

## Create Records
You can generate needed records using scaffolder module which is included into skeleton application by default, use `-f` key to specify your columns:

`spiral create:record user -f id:primary -f name:string -f email:string`
 
Once executed, locate generated `User` class in your `Database` namespace:

```php
class User extends Record
{
    const SCHEMA = [
        'id'    => 'primary',
        'name'  => 'string',
        'email' => 'string'
    ];

    const DEFAULTS = [];

    const INDEXES = [];
}
```

You can specify default values and relations to other records at this step.

## Update Schema
Now, we have to tell ORM to register our class in schems, we can do it using `orm:schema` command or `update` command:

```
> spiral orm:schema -vv
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'spiral' AND `table_name` = 'prefix_users'
[MySQLDriver] SHOW FULL COLUMNS FROM `prefix_users`
[MySQLDriver] SHOW INDEXES FROM `prefix_users`
[MySQLDriver] SELECT * FROM `information_schema`.`referential_constraints` WHERE `constraint_schema` = 'spiral' AND `table_name` = 'prefix_users'
[MySQLDriver] SHOW INDEXES FROM `prefix_users`
[MySQLDriver] SHOW TABLE STATUS WHERE `Name` = 'prefix_users'
ORM Schema have been updated: 3.825 s, records: 1, sources: 0, tables: 1, relations: 0
Table schema 'prefix_users' has changes.
Silent mode on, no databases altered.
```

By default `orm:schema` command would not alter your database schema, you have 2 options how to do that.

### Automatically alter schemas
If you wish to skip migration generation and let ORM alter your database schema automatically execute `orm:schema` command with `-a` flag.

> Please note, that this approach is quick but not recommended.

### Generate Migration
In order to gain more control on how Spiral work with your database execute `orm:schema` command with `-m` flag.

```
> spiral orm:schema -m
ORM Schema have been updated: 1.611 s, records: 1, sources: 0, tables: 1, relations: 0
Table schema 'prefix_users' has changes.
New migration created: orm-9d14973b
```

We can view our migration before running it:

```php
class MigrationOrm9d14973b extends Spiral\Migrations\Migration
{
    public function up()
    {
        $this->table('users', 'primary')
            ->addColumn('id', 'primary', [
                'nullable' => false,
                'default'  => null
            ])
            ->addColumn('name', 'string', [
                'nullable' => false,
                'default'  => '',
                'size'     => 255
            ])
            ->addColumn('email', 'string', [
                'nullable' => false,
                'default'  => '',
                'size'     => 255
            ])
            ->setPrimaryKeys([
                'id'
            ])
            ->create();
    }

    public function down()
    {
        $this->table('users', 'primary')->drop();
    }
}
```

Execute migration using `migrate` command and now you ready to work with your database.

> Use `spiral orm:schema -m && spiral migrate` to quickly model changes to your database.