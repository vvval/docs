# Constrain
Every query generated for the entity selection (directly or via relation) migth pass thought the defined contrain.
Constrains are used to define global query limits or/and filter entity by one of it's relations.

<img width="816" alt="Screenshot_76" src="https://user-images.githubusercontent.com/796136/59182959-ae1ac280-8b73-11e9-819f-d3966ef691a6.png">

In some cases you can disable constrain usage on root query to get access to unfiltered entities, use `$select->setConstrant(null)` to do that.

## Example
Simplies example can demonstrate how to only select entities which are not marked as `deleted`:

```php
class NotDeletedConstrain implements ConstrainInterface
{
    public function apply(QueryBuilder $query)
    {
        $query->where('deleted_at', '=', null);
    }
}
```

Constain can be assigned to any entity via schema, in case of `annotated` extension:

```php
/**
 * @entity(constrain="NotDeletedConstrain")
 */
class User 
{
    // ...
}
```