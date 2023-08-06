---
layout: post
title: "#3 - Laravel: Building Criteria in a Different Way"
date: 2020-06-10T09:35:00+0700
slug: laravel-building-criteria
categories: [laravel]
image: criteria.jpg
tags: [
    'backend',
    'laravel',
    'criteria',
]
---
> Filter pattern or Criteria pattern is a design pattern that enables
> developers to filter a set of objects using different criteria and chaining
> them in a decoupled way through logical operations. This type of design
> pattern comes under structural pattern as this pattern combines multiple
> criteria to obtain single criteria.

That's the definition of criteria. If we follow many tutorials, we can apply it
as follows: 

```php
public function index()
{
    $this->repository->pushCriteria(new MyCriteria1());
    $this->repository->pushCriteria(MyCriteria2::class);
    $posts = $this->repository->all();
    ...
}
class MyCriteria implements CriteriaInterface
{
    public function apply($model, RepositoryInterface $repository)
    {
        $model = $model->where('user_id', '=', Auth::user()->id);

        return $model;
    }
}
```

Or like this:

```php
$jonSnow = User::query()
    ->apply(CriteriaCollection::create([
        new CompareCriteria('name', 'Jon'),
        new CompareCriteria('lastname', 'Snow'),
    ]))
    ->first();
```

In these examples, each criterion is defined as a separate file, and we need
to call and list the criteria to use them. Although they all adhere to the
criteria pattern, it doesn't feel satisfying to me. That's why I have redefined
a new type of criterion that is unrelated to the criteria pattern. The ultimate
goal is to make the application of criteria look like this:

```php
$param = ['status' => [1, 3], 'name' => 'Reishou', 'age' => 30];

$query = User::query();
$criteria = new UserCriteria($param);
$criteria->apply($query);

$query->get();
```

with a simple `UserCriteria` file:

```php
class UserCriteria extends Criteria
{
    protected $criteria = [
        'status',
        'name' => 'like',
        'age',
    ];
}
```

The `UserCriteria` class contains the criteria that will be used for filtering,
and it will only filter the params that fall within those criteria. The crucial
part is the abstract `Criteria` class, which contains the `apply` function:

```php
public function apply($query)
{
    $this->setTable($query);
    foreach ($this->param as $key => $value) {
        $this->basicCriteria($query, $key, $value);
    }
}

protected function basicCriteria($query, $key, $value)
{
    $criteria = $this->getCriteria();

    if (in_array($key, $criteria)) {
        $value = is_array($value) ? $value : [$value];
        $query->whereIn($this->getTable() . '.' . $key, $value);

        return;
    }

    if (key_exists($key, $criteria) && Str::lower($criteria[$key]) === 'like') {
        $query->where($this->getTable() . '.' . $key, 'like', "%$value%");

        return;
    }
}
```

The `apply` function checks if the passed keys belong to the declared criteria
and applies the corresponding query functions. However, there may be cases where
we need to customize the query beyond just `whereIn` and `like`. In such cases,
we can use the custom method:

```php
protected function customMethod($query, $key, $value)
{
    $method = $this->prefixMethod . Str::studly($key);
    if (method_exists($this, $method)) {
        $this->{$method}($query, $value);

        return true;
    }

    return false;
}
public function apply($query)
{
    $this->setTable($query);
    foreach ($this->param as $key => $value) {
        if ($this->customMethod($query, $key, $value)) {
            continue;
        }
        $this->basicCriteria($query, $key, $value);
    }
}
```

If we need to customize the behavior for the `status` key, we can create the
`criteriaStatus` function in the `UserCriteria` file:

```php
protected function criteriaStatus($query, $value)
{
    $statuses = is_array($value) ? $value : [$value];
    $query->where(function ($query) use ($statuses) {
        foreach ($statuses as $status) {
            $query->orWhere($this->getTable() . '.status', $status);
        }
    });
}
```

I have written this Criteria function into a package called
[criteria-builder](https://github.com/reishou/criteria-builder).
Note the following when using the package:
- `$query` can be either `Illuminate\Database\Query\Builder` or
`Illuminate\Database\Eloquent\Builder`.
- Some params will be removed before actually executing the query, such as
objects, null, and empty arrays.

```php
protected function filterParam()
{
    return function ($value) {
        if (is_object($value)) {
            return false;
        }

        return is_array($value) ? !empty($value) : $value !== null;
    };
}
```

- You can override the `setParam` function if you don't want to remove the
aforementioned params.
- The custom method can change the default prefix.

```php
/** @var string $prefixMethod */
protected $prefixMethod = 'criteria';
```

- When creating a new criteria without passing params, by default, the params
will be taken from `request()->query()`.

```php
public function __construct(array $param = [])
{
    $param = $param ?: request()->query();
    $this->setOriginal($param);
    $this->setParam($param);
}
```

Feel free to explore the criteria-builder package for more details and usage
instructions.
