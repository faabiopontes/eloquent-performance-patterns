# Learning with Eloquent Performance Patterns

## Lesson 3

- UserController.php

```php
$users = User::query()
  ->withLastLoginAt()
  ->orderBy('name')
  ->paginate();
```

- User.php

```php
public function scopeWithLastLoginAt($query)
{
  $query->addSelect(['last_login_at' => Login::select('created_at')
    ->whereColumn('user_id', 'users.id') // links subquery with parent query
    ->latest() // by default orders by created_at DESC
    ->take(1) // limit to 1, should always be there in subquery since only one result per line
    ->withCasts(['last_login_at' => 'datetime']); // changes result from string to Carbon
```

## Lesson 4 - Dynamic Relationships Using Subqueries

- This is used to solve performance issues
- Can't be lazy loaded
- Can only be used to get one result

- UserController.php

```php
$users = User::query()
  ->withLastLogin()
  ->orderBy('name')
  ->paginate();
```

- User.php

```php
public function lastLogin()
{
  return $this->belongsTo(Login::class);
}

public function scopeWithLastLogin($query)
{
  $query->addSelect(['last_login_id' => Login::select('id')
    ->whereColumn('user_id', 'users.id')
    ->latest()
    ->take(1)
  ])->with('lastLogin');
}
```

## Lesson 5 - Calculate Totals Using Conditional Aggregates

- Problem: Table features has different status, count number of features in each status
- In SQL `SELECT count(CASE WHEN status = 'Requested' THEN 1 END) AS requested` can be used to count depending on values that were returned

```php
$statuses = Feature::toBase()
  ->selectRaw("COUNT(CASE WHEN status = 'Requested' THEN 1 END) AS requested")
  ->selectRaw("COUNT(CASE WHEN status = 'Planned' THEN 1 END) AS planned")
  ->selectRaw("COUNT(CASE WHEN status = 'Completed' THEN 1 END) AS completed")
  ->first();
```

## Lesson 6 - Optimize Circular Relationship

- Problem: Get Author of Post comment

- FeaturesController

```php
public function show(Feature $feature)
{
  $feature->load('comments.user', 'comments.feature.comments');

  // Loads relationship and assign to specific model
  // Fixes circular relationships
  $feature->comments->each->setRelation('feature', $feature);

  return view('feature', ['feature' => $feature]);
}
```

- Comment.php

```php
public function feature()
{
  return $this->belongsTo(Feature::class);
}

public function isAuthor()
{
  return $this->feature->comments->first()->user_id === $this->user_id;
}
```

## Lesson 7 - Multi-Column Searching

- Problem: Search user by First Name, Last Name, or Company Name

```php
collect(explode(' ', $terms)) // split search terms
  ->filter()
  ->each(function ($term) use ($query) {
    $term = '%'.$term.'%'; // searching like
    $query->where(function ($query) use ($term) {
      $query->where('first_name', 'like', $term)
        ->orWhere('last_name', 'like', $term)
        ->orWhereHas('company', function ($query) use ($term) {
          $query->where('name', 'like', $term);
        });
    });
  });
```

- Result: Both queries running at 326 ms (155/172ms for count/get columns)

## Lesson 8 - Getting LIKE to use an Index

- Indexes created for `first_name`, `last_name`, `company`

```php
  // Company table migration
  $table->string('name')->index();

  // Users table migration
  $table->string('first_name')->index();
  $table->string('last_name')->index();
```

- To understand what's happening in a query, like which indexes are being used, the `EXPLAIN` keyword should be used before select

- Indexes cant be used to wildcard at the beginning in MySQL

```php
  // Before
  $term = '%'.$term.'%';

  // After
  $term = $term.'%';
```

- Search groups solve multi word companies

```php
  // Before
  collect(explode(' ', $terms)) // split search terms

  // After
  // str_getcsv - Parse a CSV string into an array
  // ($string, $separator, $enclosure)
  collect(str_getcsv($terms, ' ', '"'))
```

- Result: Both queries running at 275 ms

# Lesson 9 - Faster Options Than whereHas

- First option when `whereHas` is slow try using a `join` instead

```php
  // Before
  ->orWhereHas('company', function ($query) use ($term) {
    $query->where('name', 'like', $term);
  });

  // After
  // at the top of scopeSearch
  $query->join('companies', 'companies.id', '=', 'users.company_id');

  // ...
  // replacing whereHas
  ->orWhere('companies.name', 'like', $term);
```

- Result: Both queries running at 231 ms
- Using `EXPLAIN` the indexes show as possible_keys but don't show as key being used

- Second option replace `whereHas` with `whereIn`

```php
  ->orWhereIn('company_id', function ($query) use ($term) {
    $query->select('id')
      ->from('companies')
      ->where('name', 'like', $term);
  });
```

- Result: Both queries running at 94 ms
- Always try to remove unnecessary dependencies in your query

# Lesson 10 - When it Makes Sense to Run Additional Queries

- When queries run in isolation they are able to use all the indexes
- Sometimes the query builder is unable to use all the indexes

```php
// Before (3 queries)
$query->orWhereIn('company_id', function ($query) use ($term) {
  $query->select('id')
    ->from('companies')
    ->where('name', 'like', $term);
});

// After (6 queries)
$query->orWhereIn('company_id', Company::query()
  ->where('name', 'like', $term)
  ->pluck('id')
);
```

# Lesson 11 - Use Unions to Run Queries Independently

- When you run a `UNION` query, both queries should return the same columns
- To use a `UNION` query inside a `IN` statement, we should use a derived table

```php
// Before (6 queries / 3.93ms)
$query->where(function ($query) use ($term) {
  $query->where('first_name', 'like', $term)
    ->orWhere('last_name', 'like', $term)
    ->orWhereIn('company_id', Company::query()
      ->where('name', 'like', $term)
      ->pluck('id')
    );
});

// After (3 queries / 3.84ms)
$query->whereIn('id', function ($query) use ($term) {
  $query->select('id')
    ->from(function ($query) use ($term) {
      $query->select('id')
        ->from('users')
        ->where('first_name', 'like', $term)
        ->orWhere('last_name', 'like', $term)
        ->union(
          $query->newQuery()
            ->select('users.id')
            ->from('users')
            ->join('companies', 'companies.id', '=', 'users.company_id')
            ->where('companies.name', 'like', $term)
        );
    }, 'matches');
})
```

# Lesson 12 - Fuzzier Searching With Regular Expressions

- Problem: Searching "Tim Oreilly" and "VanGogh" doesn't return any results

```php
// Before (3 queries / 3.84ms)
$term = $term.'%';
...
->where('first_name', 'like', $term)
->orWhere('last_name', 'like', $term)
...
->where('companies.name', 'like', $term)

// After (3 queries / 595ms)
$term = preg_replace('/[^A-Za-z0-9]/', '', $term).'%';
...
->whereRaw("regexp_replace(first_name, '[^A-Za-z-0-9]', '') like ?", [$term])
->orWhereRaw("regexp_replace(last_name, '[^A-Za-z-0-9]', '') like ?", [$term])
...
->orWhereRaw("regexp_replace(companies.name, '[^A-Za-z-0-9]', '') like ?", [$term])

```

- We need to create a new column to be able to use indexes
- Unless you absolutely need and index, remove it
- They cost space and computation when the column value changes

```php
  // Company table migration
  // Before
  $table->string('name')->index();

  // After
  $table->string('name'); // or ->dropIndex('name') in case it was already ran
  $table->string('name_normalized')
    ->virtualAs("regexp_replace(companies.name, '[^A-Za-z-0-9]', '')")
    ->index();

  // Users table migration
  // Before
  $table->string('first_name')->index();
  $table->string('last_name')->index();

  // After
  $table->string('first_name'); // or ->dropIndex('name') in case it was already ran
  $table->string('first_name_normalized')
    ->virtualAs("regexp_replace(companies.name, '[^A-Za-z-0-9]', '')")
    ->index();
  $table->string('last_name'); // or ->dropIndex('name') in case it was already ran
  $table->string('last_name_normalized')
    ->virtualAs("regexp_replace(companies.name, '[^A-Za-z-0-9]', '')")
    ->index();

  // User.php
  // Before (3 queries / 595ms)
  // ...
  ->whereRaw("regexp_replace(first_name, '[^A-Za-z-0-9]', '') like ?", [$term])
  ->orWhereRaw("regexp_replace(last_name, '[^A-Za-z-0-9]', '') like ?", [$term])
  // ...
  ->orWhereRaw("regexp_replace(companies.name, '[^A-Za-z-0-9]', '') like ?", [$term])

  // After (3 queries, 8 ms)
  // ...
  ->whereRaw('first_name_normalized', 'like', $term)
  ->orWhereRaw('last_name_normalized', 'like', $term)
  // ...
  ->orWhereRaw('companies.name_normalized', 'like', $term)
```

# Section 3 - Performance

## Lesson 13 - Run Authorization Policies in the Database

- If sales representative is owner, it should be able to see all customers of the company

```php
// CustomerPolicy
public function view(User $user, Customer $customer)
{
  return $user->is_owner || $user->id === $customer->sales_rep_id;
}

// CustomersController
public function index()
{
  Auth::login(User::where('name', 'Sarah Seller')->first());
}

// customers.blade
@foreach ($customers as $customer)
  @can('view', $customer)
  ...
  @endcan
@endforeach
```

- This kind of fixes the problem, but breaks pagination

```php
// CustomersController
$customers = Customer::query()
  // ...
  ->get() // this is bad because loads all users
  ->filter(function ($customer) {
    return Auth::user()->can('view', $customer);
  });

// customers.blade
```

- This loads all users and breaks pagination

```php
// CustomersController
$customers = Customer::query()
  ->with('salesRep')
  // Before
  ->get()
  // After
  ->visibleTo(Auth::user());

// Customer.php
public function scopeVisibleTo($query, User $user)
{
  if ($user->is_owner) {
    return;
  }
  $query->where('sales_rep_id', $user->id);
}
```

- Sometimes we can't do things in the Controller and have to go to the database layer with scopes
- This is a simple example, but can be used in much more complex examples