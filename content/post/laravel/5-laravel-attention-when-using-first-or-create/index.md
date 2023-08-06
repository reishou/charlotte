---
layout: post
title: "#5 - Laravel: Attention When Using firstOrCreate"
date: 2022-04-24T15:14:00+0700
slug: laravel-attention-when-using-first-or-create
categories: [laravel]
image: first-or-create.png
tags: [
    'laravel',
    'eloquent',
    'database'
]
---

## Logic

User profiles are not created during user registration. Let's write the
`getProfile($userId)` function to retrieve user profile information. If the
profile doesn't exist yet, we'll create it with default parameters and return
the profile information. Note that the `profiles` table has a unique key
constraint on `user_id`.

## Implement

To solve this, we can use the `firstOrCreate` function, which is a convenient
way to retrieve the first record matching the attributes or create it if
it doesn't exist.

```php
$profile = Profile::firstOrCreate(
    ['user_id' => $userId],
    $this->getDefaultProfile()
);
```

## Life is not easy 

However, life is not always that simple. After testing locally, committing,
pushing, and deploying, you suddenly receive errors from the database via Sentry:

```text
Illuminate\Database\QueryException: SQLSTATE[23505]: Unique violation: 7 ERROR:
duplicate key value violates unique constraint "profiles_user_id_unique"
DETAIL:  Key (user_id)=(45661934453770) already exists. (SQL: insert into "profiles"...
```

Upon checking the `profiles` table, you find that a record with 
`user_id=45661934453770` already exists. You wonder how to insert when the
record already exists, given the behavior of the `firstOrCreate` function.

```php
# vendor/laravel/framework/src/Illuminate/Database/Eloquent/Builder.php

public function firstOrCreate(array $attributes = [], array $values = [])
{
    if (! is_null($instance = $this->where($attributes)->first())) {
        return $instance;
    }

    return tap($this->newModelInstance($attributes + $values), function ($instance) {
        $instance->save();
    });
}
```

While contemplating where to start fixing the bug, you decide to check the
Nginx log:

```
[24/Apr/2022:07:52:04 +0000] "GET /v1/users/45661934453770/profile HTTP/1.1" 200
[24/Apr/2022:07:52:04 +0000] "GET /v1/users/45661934453770/profile HTTP/1.1" 500
```

Aha! You've got it. There were two simultaneous requests, and both requests
evaluated `is_null($instance = $this->where($attributes)->first())` as `true`.
However, when `$instance->save()` was called, only one of the two requests
succeeded, resulting in the error for the other request.

## Solution

Instead of using the `firstOrCreate` function and considering transactions and
table locks, which may not be suitable for systems with many simultaneous
requests, I suggest using the `upsert` function introduced in Laravel version 8.10. 

```php
public function getProfile($userId)
{
    $profile = $this->getFirst($userId);
    
    if ($profile) {
        return $profile;
    }
    
    Profile::upsert($this->getDefaultProfile(), ['user_id']);
    
    return $this->getFirst($userId);
}

protected function getFirst($userId)
{
    return Profile::where('user_id', $userId)->first();
}
```

If you're using Laravel version prior to 8.10, you can use an alternative
package like [laravel-upsert](https://github.com/staudenmeir/laravel-upsert).
When two requests come simultaneously, the `upsert` function will not throw an
error, even if the `profiles` table already contains a record for that user ID.
The behavior of the `upsert` function may vary depending on the database
management system being used. The following is an example of the underlying
SQL statement (for MySQL), but keep in mind that the actual command may differ
for different database systems:

```sql
INSERT INTO table (column_list)
VALUES (value_list)
ON DUPLICATE KEY UPDATE
   c1 = v1, 
   c2 = v2,
   ...;
```

## Conclusion

In cases where duplicate entries are acceptable and there is no unique key
constraint, you can continue using firstOrCreate as usual. However, I recommend
limiting its use and considering other solutions like upsert when appropriate.
