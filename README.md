[![Build Status](https://travis-ci.org/staudenmeir/eloquent-json-relations.svg?branch=master)](https://travis-ci.org/staudenmeir/eloquent-json-relations)
[![Latest Stable Version](https://poser.pugx.org/staudenmeir/eloquent-json-relations/v/stable)](https://packagist.org/packages/staudenmeir/eloquent-json-relations)
[![Total Downloads](https://poser.pugx.org/staudenmeir/eloquent-json-relations/downloads)](https://packagist.org/packages/staudenmeir/eloquent-json-relations)
[![License](https://poser.pugx.org/staudenmeir/eloquent-json-relations/license)](https://packagist.org/packages/staudenmeir/eloquent-json-relations)

## Introduction
This Laravel Eloquent extension adds support for JSON foreign keys to `BelongsTo`, `HasOne`, `HasMany`, `HasManyThrough`, `MorphTo`, `MorphOne` and `MorphMany` relationships.  
It also provides [many-to-many](#many-to-many-relationships) relationships with JSON arrays.

## Compatibility

 Database         | Laravel
:-----------------|:----------
 MySQL 5.7+       | 5.5.29+
 MariaDB 10.2+    | 5.8+ (2019)
 PostgreSQL 9.3+  | 5.5.29+
 [SQLite 3.18+](https://www.sqlite.org/json1.html) | 5.6.35+
 SQL Server 2016+ | 5.6.25+
 
## Installation

    composer require staudenmeir/eloquent-json-relations

## Usage

   * [Referential Integrity](#referential-integrity)
   * [Many-To-Many Relationships](#many-to-many-relationships)

In this example, `User` has a `BelongsTo` relationship with `Locale`. There is no dedicated column, but the foreign key (`locale_id`) is stored as a property in a JSON field (`users.options`):

```php
class User extends Model
{
    use \Staudenmeir\EloquentJsonRelations\HasJsonRelationships;

    protected $casts = [
        'options' => 'json',
    ];

    public function locale()
    {
        return $this->belongsTo('App\Locale', 'options->locale_id');
    }
}

class Locale extends Model
{
    use \Staudenmeir\EloquentJsonRelations\HasJsonRelationships;

    public function users()
    {
        return $this->hasMany('App\User', 'options->locale_id');
    }
}
```

Remember to use the `HasJsonRelationships` trait in both the parent and the related model. 

**Limitations:** Existence queries (`Locale::has('users')`) and `HasManyThrough` relationships don't work on PostgreSQL with integer keys.

### Referential Integrity

[MySQL](https://dev.mysql.com/doc/refman/en/create-table-foreign-keys.html) and [SQL Server](https://docs.microsoft.com/en-us/sql/relational-databases/tables/specify-computed-columns-in-a-table) support foreign keys on JSON columns with generated/computed columns.

Laravel migrations support this feature on MySQL:

```php
Schema::create('users', function (Blueprint $table) {
    $table->increments('id');
    $table->json('options');
    $locale_id = DB::connection()->getQueryGrammar()->wrap('options->locale_id');
    $table->unsignedInteger('locale_id')->storedAs($locale_id);
    $table->foreign('locale_id')->references('id')->on('locales');
});
```

On SQL Server, the migration requires raw SQL:

```php
Schema::create('users', function (Blueprint $table) {
    $table->increments('id');
    $table->json('options');
});

$locale_id = DB::connection()->getQueryGrammar()->wrap('options->locale_id');
DB::statement('ALTER TABLE [users] ADD "locale_id" AS CAST('.$locale_id.' AS INT) PERSISTED');

Schema::table('users', function (Blueprint $table) {
    $table->foreign('locale_id')->references('id')->on('locales');
});
```

### Many-To-Many Relationships

This package also introduces two new relationship types: `BelongsToJson` and `HasManyJson`

On Laravel 5.6.25+, you can use them to implement many-to-many relationships with JSON arrays.

In this example, `User` has a `BelongsToMany` relationship with `Role`. There is no pivot table, but the foreign keys (`role_ids`) are stored as an array in a JSON field (`users.options`):

```php
class User extends Model
{
    use \Staudenmeir\EloquentJsonRelations\HasJsonRelationships;

    protected $casts = [
       'options' => 'json',
    ];
    
    public function roles()
    {
        return $this->belongsToJson('App\Role', 'options->role_ids');
    }
}

class Role extends Model
{
    use \Staudenmeir\EloquentJsonRelations\HasJsonRelationships;

    public function users()
    {
       return $this->hasManyJson('App\User', 'options->role_ids');
    }
}
```

On the side of the `BelongsToJson` relationship, you can use `attach()`, `detach()`, `sync()` and `toggle()`:

```php
$user = new User;
$user->roles()->attach([1, 2]);
$user->save();

$user->roles()->detach([2])->save(); // Now: [1]

$user->roles()->sync([1, 3])->save(); // Now: [1, 3]

$user->roles()->toggle([2, 3])->save(); // Now: [1, 2]
```

**Limitations:** These relationships don't work on SQLite.