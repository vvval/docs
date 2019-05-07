# Has One
The Has Many relation defines that entity exclusively owns multiple other entities in a form of parent-children.

## Definition
To define Has Many relation using annotated enties extension use:

```php
/** @entity */ 
class User 
{
    // ...
    
    /** @hasMany(target = "Post") */
    protected $posts;
}
```

In order to use newly created entity you must define the collection to store related entities. Do it in your constructor:

```php
use use Doctrine\Common\Collections\ArrayCollection;

/** @entity */ 
class User 
{
    // ...
    
    /** @hasMany(target = "Post") */
    protected $posts;

    public function __construct()
    {
        $this->address = new ArrayCollection();
    }
       
    // ...
    
    public function getPosts()
    {
        return $this->posts;
    }
}
```

By default, ORM will generate outer key in relation object using parent entity role and inner key (primary key by default) values. As result column and FK will be added to Address entity on `user_id` column.

Option      | Value  | Comment
---         | ---    | ----
cascade     | bool   | Automatically save related data with parent entity, defaults to `true`
nullable    | bool   | Defines if relation can be nullable (child can have no parent), defaults to `false`
innerKey    | string | Inner key in parent entity, defaults to primary key
outerKey    | string | Outer key name, defaults to `{parentRole}_{innerKey}`
where       | array  | Additional where condition to be applied for the relation, defaults to none.
fkCreate    | bool   | Set to true to automatically create FK on outerKey, defauls to `true`
fkAction    | CASCADE, NO ACTION, SET NULL | FK onDelete and onUpdate action, defaults to `CASCADE`  
indexCreate | bool   | Create index on outerKey, defaults to `true`

## Usage
To add the child object to the collection, use collection method `add`:

```php
$u = new User();
$u->getPosts()->add(new Post("test post"));
```

The related object(s) can be immediate saved into the database by persisting parent entity:

```php
$t = new Transaction($orm);
$t->persist($u);
$t->run();
```

To delete previously associated object call `remove` or `removeElement` methods of the collection:

```php
$post = $u->getPosts()->get(0);
$u->getPosts()->removeElement(post);
```

The child object will be removed during the persist operation.

> Set relation option `nullable` as true to nullify the outer key istead of entity removal.

## Loading
To access related data simply call the method `load` of your `User` `Select` object:

```php
$users = $orm->getRepository(User::class)
    ->select()
    ->load('posts')
    ->wherePK(1)
    ->fetchAll();

foreach ($users as $u) {  
    print_r($u->getPosts());
}
```

Please note, by default ORM will load HasMany related entities using external query (`WHERE IN`).

## Filtering
You can filter entity selection using related data, call method `with` of your entity `Select` to join related entity table:

```php
$users = $orm->getRepository(User::class)
    ->select()
    ->distinct()
    ->with('posts')->where('posts.published', true)
    ->fetchAll();
    
print_r($users);
```

> Make sure to call `distinct` since multi-row table will be joined to your query.

Cycle `Select` can automatically join related table on first `where` condition, previous example can be rewritten:

```php
$users = $orm->getRepository(User::class)
    ->select()
    ->distinct()
    ->where('posts.published', true)
    ->fetchAll();
    
print_r($users);
```

## Load filtered
Another option available for HasMany relation is to pre-filter related data on database level. To do that, use `where` option of the 
relation or `load` method. For example we can load all users with at least one post and pre-load only published posts:

```php
$users = $orm->getRepository(User::class)
    ->select()
    ->distinct()
    ->with('posts')
    ->load('posts', ['where' => ['published' => true]])
    ->fetchAll();
```

Another option is to use the `with` selection to drive the data for the pre-loaded entities. You can point your `load` method to use 
`with` filtered relation data via `using` flag:

```php
$users = $orm->getRepository(User::class)
    ->select()
    ->distinct()
    ->with('posts', ['as' => 'published_posts'])->where('posts.published', true)
    ->load('posts',['using' => 'published_posts'])
    ->fetchAll();
```

Given appoarch will produce only one SQL query.

```sql
SELECT DISTINCT
  `user`.`id` AS `c0`, 
  `user`.`name` AS `c1`, 
  `published_posts`.`id` AS `c2`, 
  `published_posts`.`title` AS `c3`, 
  `published_posts`.`published` AS `c4`, 
  `published_posts`.`user_id` AS  `c5`, 
FROM `spiral_users` AS `user`
INNER JOIN `spiral_posts` AS `published_posts`
    ON `published_posts`.`user_id` = `user`.`id`
WHERE `published_posts`.`published` = true
```

You can also pre-set the conditions in the relation definition:

```php
class User 
{
    // ...
    
    /** @hasMany(target = "Post", where={"published": true}) */
    protected $posts;

    public function __construct()
    {
        $this->address = new ArrayCollection();
    }
       
    // ...
    
    public function getPosts()
    {
        return $this->posts;
    }
}
```