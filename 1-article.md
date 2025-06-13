# Managing Hierarchical Data in Laravel: Recursive, Adjacency List, and Nested Set Compared

Working with hierarchical data — such as categories, menus, or threaded comments — is a common requirement in many
Laravel applications.
Whether you're building an e-commerce catalog, managing file systems,
or handling complex organizational structures, you need an efficient way to store and retrieve tree-like relationships
in your database.

In this article, we'll explore **three different strategies** for managing hierarchical category data in Laravel 12
APIs:

1. Laravel's built-in **recursive relationship**
2. The **adjacency list with CTE** approach using [
   `staudenmeir/laravel-adjacency-list`](https://github.com/staudenmeir/laravel-adjacency-list)
3. The classic **nested set model** using [`kalnoy/nestedset`](https://github.com/lazychaser/laravel-nestedset)

We'll demonstrate these with a real-world example: a 1000-node category tree for an automotive parts catalog, benchmark
each method, and discuss when each approach is most suitable.

## The Schema: Categories Table

All three approaches share the same base table structure:

```php
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug', 100)->unique();
    $table->foreignId('parent_id')->nullable()->constrained('categories');
    $table->timestamps();
});
```

This classic structure is ideal for building recursive relationships and works well with both packages we’ll be using.

We seeded this table with a realistic category dataset, for example:

```
Auto Parts & Components
└── Engine Parts
    └── Engine Bearings
Performance & Tuning
└── Performance Engine Parts
    └── Performance Valve Springs
Tools & Equipment
└── Hand Tools
    ├── Sockets
    └── Tap and Die Sets
Vehicle Systems
├── Exhaust Management System
│   └── Fleet Series for Bentley RV
└── Lighting System
    └── Acura Economy Series
Vehicle Types
└── Truck
    └── Mercedes-Benz RV Japanese Options
Electronics & Technology
└── Gauges & Monitors
    └── Volvo Minivan American Options
Tires & Wheels
└── SUV Tires
    └── Nissan Sedan Professional Options
Maintenance & Repair
└── Undercoating
    └── Chrysler Mid-Range Essentials
```


![Database Categories](assets/db-categories.jpg)

We exposed the following endpoints in the API:

```php
Route::prefix('catalog/categories')->group(function () {
    Route::get('tree-recursive', [CategoryController::class, 'treeRecursive']);
    Route::get('tree-adjacency', [CategoryController::class, 'treeAdjacency']);
    Route::get('tree-nested-set', [CategoryController::class, 'treeNestedSet']);
});
```

![Endpoint Output in Postman](assets/category-tree-output-postman.jpg)

## Approach 1: Recursive Relationship

Laravel natively supports recursive relationships through Eloquent.
You can define a `children()` relation recursively:

```php
final class Category extends Model
{
    public function children(): HasMany
    {
        return $this->hasMany(Category::class, 'parent_id')->with('children');
    }
}
```

![Category model relationship](assets/category-model-relationship.jpg)

To fetch the entire tree:

```php
final class ListCategoryTreeRecursiveAction implements Actionable
{
    public function handle(): Collection
    {
        return Category::query()
            ->whereNull('parent_id')
            ->with('children')
            ->get();
    }
}
```

This triggers multiple SQL queries (one per depth level):

```
select * from categories where parent_id is null;
select * from categories where parent_id in (...);
select * from categories where parent_id in (...);
...
```

![Approach 1: queries](assets/approach-1-queries.jpg)

**Pros:**

* Native to Laravel, no packages required.
* Great for shallow or medium-depth trees.

**Cons:**

* N+1 query problem for deep trees.
* Slower performance as depth grows.

## Approach 2: Adjacency List (CTE)

[staudenmeir/laravel-adjacency-list](https://github.com/staudenmeir/laravel-adjacency-list) simplifies recursive queries
using Common Table Expressions (CTEs) for performance.

Install the package:
`composer require staudenmeir/laravel-adjacency-list`

Add the trait to your model:

```php
use Staudenmeir\LaravelAdjacencyList\Eloquent\HasRecursiveRelationships;

final class Category extends Model
{
    use HasRecursiveRelationships;
}
```

Tree loading:

```php
final class ListCategoryTreeAdjacencyAction implements Actionable
{
    public function handle(): Collection
    {
        return Category::tree()->get()->toTree();
    }
}
```

Under the hood, it generates a recursive CTE:

```sql
with recursive `laravel_cte` as (select *
   from categories
   where parent_id is null
   union all
   select categories.*
   from categories inner join laravel_cte on laravel_cte.id = categories.parent_id)
select * from laravel_cte;
```

This approach offloads the recursion to the **database engine**, reducing the number of queries to one.

![Approach 2: queries](assets/approach-2-queries.jpg)

**Pros:**

* Efficient single-query fetch.
* Built-in depth and path tracking.
* More maintainable than raw recursion.
* Other handy methods for working with recursive relationships.

**Cons:**

* Requires package.
* Slight overhead over native recursion (but more scalable).

## Approach 3: Nested Set Model

Nested Set stores hierarchical data with `_lft` and `_rgt` boundaries, enabling the retrieval of full trees with a
single flat query.

The [nested set model](https://en.wikipedia.org/wiki/Nested_set_model) precomputes hierarchy into `_lft` and `_rgt`
columns for **blazing-fast reads**, at the cost of more complex writes.

```bash
composer require kalnoy/nestedset
```

Add columns in a migration:

```php
Schema::table('categories', function (Blueprint $table) {
    $table->unsignedInteger('_lft')->default(0)->after('parent_id');
    $table->unsignedInteger('_rgt')->default(0)->after('_lft');
});
```

Use a separate model to avoid conflicts:

```php
use Kalnoy\Nestedset\NodeTrait;

final class CategoryNode extends Model
{
    use NodeTrait;

    protected $table = 'categories';
}
```

Don’t forget to initialize the tree:

```php
(new CategoryNode())->newNestedSetQuery()->fixTree();
```

`fixTree()` recalculates the `_lft` and `_rgt` values and should be run after bulk inserts or when the structure becomes inconsistent.

Fetch tree:

```php
final class ListCategoryTreeNestedSetAction implements Actionable
{
    public function handle(): Collection
    {
        /** @var Collection $categories */
        $categories = CategoryNode::query()->get();

        return $categories->toTree();
    }
}
```

SQL:

```sql
select * from categories;
```

Just a **single query**, with all hierarchical logic resolved via `_lft`/`_rgt`.

**Pros:**

* Best performance for read-heavy scenarios.
* Single flat query, minimal joins.

**Cons:**

* More complex write operations (inserts/moves).
* Requires careful tree rebuilding after large imports.

## Benchmark Results

We benchmarked each approach by running each 100 times with the 1,000-category tree:

```php
$measures = Benchmark::measure([
	class_basename(ListCategoryTreeRecursiveAction::class) => fn() => $listCategoryTreeRecursiveAction->handle(),
	class_basename(ListCategoryTreeAdjacencyAction::class) => fn() => $listCategoryTreeAdjacencyAction->handle(),
	class_basename(ListCategoryTreeNestedSetAction::class) => fn() => $listCategoryTreeNestedSetAction->handle(),
], 100);

$this->table(array_keys($measures), [$measures]);
```

### Result

Environment: Laravel 12, PHP 8.2, MySQL 8.0 on Intel i5 (16GB RAM)

| Recursive   | Adjacency List | Nested Set      |
|-------------|----------------|-----------------|
| 25.945174ms | 29.596162ms    | **22.014018ms** |

The results were averaged over 100 runs; performance may vary based on hardware, dataset size, and SQL optimizations.

![Benchmark result](assets/benchmark.jpg)

![Benchmark chart](assets/benchmark-chart.jpg)

> The nested set model was the fastest due to minimal computation at runtime. The recursive approach was not far behind,
> while the adjacency list (despite using CTE) had a slight overhead.

## When to Use Each Approach?

| Approach        | Pros                                      | Cons                                       | Use When...                                  |
|-----------------|-------------------------------------------|--------------------------------------------|----------------------------------------------|
| Recursive       | Native to Laravel, simple to implement    | Multiple queries, slower for deep trees    | You have limited depth and want zero setup   |
| Adjacency (CTE) | Single query, DB-level recursion          | Requires package, slower in PHP processing | You want SQL-powered trees with decent depth |
| Nested Set      | Blazing fast reads, ideal for large trees | Harder to maintain, complex updates        | Your data is read-heavy and rarely updated   |

If you’re dealing with a **small, manageable set of categories**, Laravel’s native recursion is more than enough—simple
and expressive.

When you **need scalability**, especially for **deep hierarchies**, consider the **Adjacency List**. It offers a balance
between simplicity and performance.

For **maximum read performance**, particularly when rendering large menus or exporting full trees, **Nested Set**
wins—but beware of its complexity when modifying the structure.

## Source Code

You can explore the full implementation and examples shown in this article on GitHub:
👉 **[tegos/laravel-hierarchical-data](https://github.com/tegos/laravel-hierarchical-data)**

The repository includes:

* Example category data seeder
* Actions for each tree-building approach
* API routes and controller logic
* Benchmarking scripts

## Summary Table

| Approach   | Setup Effort | Read Perf. | Write Perf. | Ideal Use Case         |
|------------|--------------|------------|-------------|------------------------|
| Recursive  | Low          | Medium     | High        | Small trees            |
| Adjacency  | Medium       | Medium     | Medium      | Medium-depth trees     |
| Nested Set | High         | High       | Low         | Read-heavy, deep trees |


## Resources

* [Laravel relationships](https://laravel.com/docs/master/eloquent-relationships)
* [Recursive hasMany Relationship](https://laraveldaily.com/post/eloquent-recursive-hasmany-relationship-with-unlimited-subcategories)
* [Recursive Nested Data](https://codecourse.com/articles/recursive-nested-data-in-laravel)
* [What Is a Recursive CTE in SQL?](https://learnsql.com/blog/sql-recursive-cte)
* [Optimizing Tree Data in SQL: The Nested Set Model](https://medium.com/@leo.dorn/optimizing-tree-data-in-sql-the-nested-set-model-for-tree-structures-72c1aae0f6f9)


## Final Thoughts

Hierarchical data structures are notoriously tricky to manage efficiently. Laravel offers a lot of flexibility, whether
you're working with built-in recursive relationships, SQL-driven CTEs, or performance-optimized nested sets.

Each strategy has its place:

* **Go recursive** when your tree is small and you want quick setup.
* **Use adjacency lists** for medium trees and clear SQL recursion.
* **Choose nested sets** for massive or performance-critical read scenarios.

There’s no one-size-fits-all. Choose the strategy that best fits your data structure, usage patterns, and performance needs.

