Laravel’s SoftDelete is the combination of two things 

1.  A global scope to limit soft deleted records.
2.  An overridden eloquent’s delete method to insert fresh timestamps instead of actually deleting the records.

You see how simple is that, it just ask eloquent two things and these are

### 1) Global Scope

The first thing it asks is, hey eloquent, whenever you want to retrieve any record from the database, you are going to add just one more ‘where condition’ to the query. And that condition is _whereNull(DELETED_AT)_, which means just retrieve only those records whose _deleted_at_ column is null (non soft-deleted records)

### 2) Overridden Delete Method

And the second thing it asks is, hey eloquent, whenever you want to delete any record, you are going to call my delete method instead of your default delete method. What I am going to do in this method is, instead of actually deleting records, I will put fresh timestamps at _deleted_at_ column.

## Code Demonstration

Let's have a look at the code and see how these things are actually implemented in there.

So whenever we want to soft-delete our records, we use _SoftDeletes_ trait in our model which is _Illuminate\Database\Eloquent\SoftDeletes_, right! Let's start from _SoftDeletes_ trait and see what’s going on there.

There are many methods you will see but the most important method is _bootSoftDeletes()_.

File: _Illuminate\Database\Eloquent\SoftDeletes.php_

```php public static function bootSoftDeletes(){ static::addGlobalScope(new SoftDeletingScope); } ```

This method adds global scope to the modal which does the rest of the job. This method is called by eloquent whenever eloquent is created. If you wanna see how it gets called then let's jump to constructor of the eloquent modal.

File: _Illuminate\Database\Eloquent\Model.php_

```php public function __construct(array $attributes = []){ $this->bootIfNotBooted(); $this->syncOriginal(); $this->fill($attributes); } ```

The method of our interest is _bootIfNotBooted()_.

File: _Illuminate\Database\Eloquent\Model.php_

```php protected function bootIfNotBooted(){ if (! isset(static::$booted[static::class])) { static::$booted[static::class] = true; $this->fireModelEvent('booting', false); static::boot(); $this->fireModelEvent('booted', false); } } ```

the method of our concern here is _static::boot()_.

```php protected static function boot(){ static::bootTraits(); } protected static function bootTraits(){ $class = static::class; foreach(class_uses_recursive($class) as $trait) { if(method_exists( $class$method='boot'.class_basename($trait))) { forward_static_call([$class, $method]); } } } ```

In brief, this method calls function of every trait in the modal whose name matches bootTraitName type signature i.e. bootMagicTrait for MagicTrait.

In our case of _SoftDeletes_ trait, it is _bootSoftDeletes()_;

File: _Illuminate\Database\Eloquent\SoftDeletes.php_

```php public static function bootSoftDeletes(){ static::addGlobalScope(new SoftDeletingScope); } ```

So this is the boot method of our _SoftDeletes_ trait and it’s just adding a global scope _SoftDeletingScope_ which we've already discussed.

Now it's time to jump to _SoftDeletingScope_.

```php class SoftDeletingScope implements Scope{ public function apply(Builder $builder, Model $model){ $builder->whereNull($model->getQualifiedDeletedAtColumn()); } public function extend(Builder $builder){ foreach ($this->extensions as $extension) { $this->{"add{$extension}"}($builder); } $builder->onDelete(function (Builder $builder) { $column = $this->getDeletedAtColumn($builder); return $builder->update([ $column => $builder->getModel()->freshTimestampString(), ]); }); } //extend function ends here } //class ends here ```

The two important methods here are **extends** and **apply**. Eloquent calls both methods at some point of the execution. 

In brief words, the **extend** method is called whenever we build new query for the model (it's time when eloquent apply global scopes) and **apply** method is called whenever we retrieve new records through model.

### **Extend method:**

Whenever we build query for any model, function _newQuery()_ is called.

File: _Illuminate\Database\Eloquent\Model.php_

```php public function newQuery(){ $builder = $this->newQueryWithoutScopes(); foreach ($this->getGlobalScopes() as $identifier => $scope) { $builder->withGlobalScope($identifier, $scope); } return $builder; } ```

This calls _withGlobalScope_ method.

File: _Illuminate\Database\Eloquent\Builder.php_

```php public function withGlobalScope($identifier, $scope){ $this->scopes[$identifier] = $scope; if (method_exists($scope, 'extend')) { $scope->extend($this); } return $this; } ```

and this method calls **_extend_** method of the scope given to it. 

Let's jump to our extend method of _SoftDeletingScope_.

File: _Illuminate\Database\Eloquent\_SoftDeletingScope_.php_

```php public function extend(Builder $builder){ foreach ($this->extensions as $extension) { $this->{"add{$extension}"}($builder); } $builder->onDelete(function (Builder $builder) { $column = $this->getDeletedAtColumn($builder); return $builder->update([ $column => $builder->getModel()->freshTimestampString(), ]); }); } ```

And in our extend method of _SoftDeletingScope_, we do two things. 

In the first part of of method, it just add some methods (macros) to query builder which you can call for specific purpose like _forceDelete_.

### Builder's onDelete method

The second and most important thing is, we are overriding default delete method with our own method, which will be get called whenever any record is being deleted.

File: _Illuminate\Database\Eloquent\_SoftDeletingScope_.php_

```php $builder->onDelete(function (Builder $builder) { $column = $this->getDeletedAtColumn($builder); return $builder->update([ $column => $builder->getModel()->freshTimestampString() ]); }); ```

What we are doing here is instead of actually deleting any record, we are just updating _deleted_at_ column with fresh timestamps. 

### Apply method: 

This method is called whenever we try to retrieve new record from the database.

It's actually gets called from eloquent’s get method which is called by model's data retrieving methods like first, find, all or by user himself after any where clause.

File: _Illuminate\Database\Eloquent\Builder.php_

```php public function get($columns = ['*']){ $builder = $this->applyScopes(); $models = $builder->getModels($columns); if (count($models) > 0) { $models = $builder->eagerLoadRelations($models); } return $builder->getModel()->newCollection($models); } ```

this method calls _applyScopes_

File: _Illuminate\Database\Eloquent\Builder.php_

```php public function applyScopes(){ if (! $this->scopes) { return $this; } $builder = clone $this; foreach ($this->scopes as $scope) { $builder->callScope(function (Builder $builder) use ($scope) { if ($scope instanceof Closure) { $scope($builder); } elseif ($scope instanceof Scope) { $scope->apply($builder, $this->getModel()); } }); } return $builder; } ```

The code of our interest is $scope->apply, which call apply method of specific scope.

And what our _apply_ method does is, add a where clause to the query builder to restrict soft deleted columns.

File: _Illuminate\Database\Eloquent\_SoftDeletingScope_.php_

```php public function apply(Builder $builder, Model $model){ $builder->whereNull($model->getQualifiedDeletedAtColumn()); } ```

And what our apply method does is, add a where clause to the query builder to restrict soft deleted columns.

Happy coding :)
