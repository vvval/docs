# Quick Start
This guide provides quick overview of ORM installation, configuration process and example using an annotateed entity. Other sections of the documentation will provide deeper insign of various use-cases.

## Requirements
  * PHP7.1+
  * PHP-PDO
  * PDO drivers for desired databases
  
## Installation
Cycle ORM is available as composer repository and can be installed using the following command in a root of your project:

```bash
$ composer require cycle/orm
```

In order to enable support for annotated entities you have to request additional package:

```bash
$ composer require cycle/annotated
```

This command will also download Cycle ORM dependencies such as `spiral/database`, `doctrine/collections` and `zendframework/zend-hydrator`.

In order to access Cycle ORM classes make sure to include `vendor/autoload.php` in your file.

```php
<?php declare(strict_types=1);
include 'vendor/autoload.php';
```

## Configuration
In order to operate, Cycle ORM require proper database connection to be set. All database connections are managed using `DatabaseManager` service provided by the package `spiral/database`. We can configure our first database connection to be initiated on demand using the following configuration:

```php
<?php declare(strict_types=1);
include 'vendor/autoload.php';

use Spiral\Database;

$dbal = new Database\DatabaseManager(new Database\Config\DatabaseConfig([
    'default'     => 'default',
    'databases'   => [
        'default' => [
            'connection' => 'sqlite'
        ]
    ],
    'connections' => [
        'sqlite' => [
            'driver'  => Database\Driver\SQLite\SQLiteDriver::class,
            'options' => [
                'connection' => 'sqlite:database.db',
                'username'   => '',
                'password'   => '',
            ]
        ]
    ]
]));
```

> Read about how to connect to other database types in [next section](/basic/connection.md). You can also configure
database connections at runtime.

Check database access using following code:

```php
print_r($dbal->database('default')->getTables());
```

> Run `php {filename}.php`, the result must be empty array.

## ORM
Initiate ORM service:

```php
$orm = new ORM\ORM(new ORM\Factory($dbal));
```

> Make sure to add `use Cycle\ORM;` at top of your file.

## Register Namespace
We can create our first entity in a directory `src` of our project.

Register new namespace in your composer.json file:

```json
"autoload": {
    "psr-4": {
      "Example\\": "src/"
    }
  }
```

Execute: 

```bash
$ composer dump
```

## Define Entity
To create our first entity in `src` folder we will use capabilities provided by `cycle/annoted` package to describe desired schema:

```php
<?php declare(strict_types=1);

namespace Example;

/**
 * @entity
 */
class User
{
    /**
     * @column(type=primary)
     * @var int
     */
    protected $id;

    /**
     * @column(type=string)
     * @var string
     */
    protected $name;
    
    public function getId(): int
    {
        return $this->id;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function setName(string $name): void
    {
        $this->name = $name;
    }
}
```

Cycle will automatically assign role `user` and table `users` from default database to this entity.

> Attention, `@entity` annotation is required!

## Schema Generation
In order to operate we need to generate ORM Schema which will describe how our entities are configured. Thought we can do it manually 
we will use the pipeline generator provided by `cycle/schema-builder` package and generators from `cycle/annotated`.

First, we have to create instance of `ClassLocator` which will automatically find needed entities:

```php
$classLocator = (new \Spiral\Tokenizer\Tokenizer(new \Spiral\Tokenizer\Config\TokenizerConfig([
    'directories' => ['src/'],
])))->classLocator();
```

We can immediatelly check if our class visible (ClassLocator will perform static indexation of your code behind the hood):

```php
print_r($classLocator->getClasses());
```

Once class locator is established we can create our schema generation pipeline. First, we will add needed namespace imports:

```php
use Cycle\Schema;
use Cycle\Annotated;
```

Now we can define our pipeline:

```php
$schema = (new Schema\Compiler())->compile(new Schema\Registry($dbal), [
    new Annotated\Entities($classLocator),    // register annotated entities
    new Schema\Generator\ResetTables(),       // re-declared table schemas (remove columns)
    new Schema\Generator\GenerateRelations(), // generate entity relations
    new Schema\Generator\ValidateEntities(),  // make sure all entity schemas are correct
    new Schema\Generator\RenderTables(),      // declare table schemas
    new Schema\Generator\RenderRelations(),   // declare relation keys and indexes
    new Schema\Generator\SyncTables(),        // sync table changes to database
    new Schema\Generator\GenerateTypecast(),  // typecast non string columns
]);
```

> We will explain what each generator is doing in a later sections. Please note, while complining your schema `SyncTables` will automatically adjust your database structure! Do not use it on real database!

The resulted schema can be passed to ORM. 

```php
$orm = $orm->withSchema(new ORM\Schema($schema));
```

> Generated schema is intended to be cached in your application, only re-generate schema when it's needed.

Your ORM is ready for use. 

> You can dump `schema` variable to check the internal representation of your entity schema.

## Persist Entity
Now, we can init and persist our first entity in database:

```php
$u = new \Example\User();
$u->setName("Hello World");
```

To persist our entity we have register it in the transaction:

```php
$t = new ORM\Transaction($orm);
$t->persist($u);
$t->run();
```

You can immediatelly dump the object to see newly generated primary key:

```php
print_r($u);
```

## Select Entity
You can select the entity from database using it's primary key and associated repository:

```php
$u = $orm->getRepository(\Example\User::class)->findByPK(1);
```

> Remove the code from the section above to avoid fetching entity from memory.

## Update Entity
To update entity data simply change it's value before persisting it in the transaction:

```php
$u = $orm->getRepository(\Example\User::class)->findByPK(1);
print_r($u);

$u->setName("New " . mt_rand(0, 1000));

(new ORM\Transaction($orm))->persist($u)->run();
```

You can notice that new name will be displayed on every script iteration.

## Delete Entity
To delete entity simply call method `delete` of the Transation:

```php
(new ORM\Transaction($orm))->delete($u)->run();
```

## Update Entity Schema
You can modify your entity schema to add new columns. Note that you have to either specify default value or set column as `nullable` in order to apply modification to the non empty table.

```php
/**
* @column(type=int,nullable=true)
* @var int|null
*/
protected $age;
```

Schema will be automatically updated on next script invocation. We can find all users with non defined age using following method:

```php
$users = $orm->getRepository(\Example\User::class)->findAll(['age' => null]);
print_r($users);
```
