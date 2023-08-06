---
layout: post
title: "#2 - Laravel: Stop Abusing try-catch in Controllers"
date: 2020-05-24T10:45:00+0700
slug: laravel-stop-abusing-try-catch
categories: [laravel]
image: try-catch.jpg
tags: [
    'backend',
    'laravel',
    'exceptions',
]
---
When reviewing the source code of other developers, I have noticed a common practice of using try-catch
blocks excessively in Laravel controllers, like the following example:

```php
public function index()
{
    try {
        return $this->success($this->userService->listAvatar());
    } catch (Exception $e) {
        Log::error('API_GET_AVATAR_ERROR =>' . $e->getMessage());
        return $this->error(trans('messages.error_common'), Response::HTTP_INTERNAL_SERVER_ERROR);
    }
}
```

However, in most cases, we don't actually need to use try-catch blocks in
controllers because Laravel already handles errors and exceptions for us.
You can find more details about Laravel's error handling in the
[Laravel documentation on errors](https://laravel.com/docs/errors).

> When you start a new Laravel project, error and exception handling is 
> already configured for you. The `App\Exceptions\Handler` class is where
> all exceptions triggered by your application are logged and then rendered back to the user.

With that in mind, we can move the error logging and handling logic to the
`render()` function within the `App\Exceptions\Handler` file.

```php
public function render($request, Exception $exception)
{
    if ($exception instanceof ResourceNotFound) {
        if ($this->needsJson($request)) {
            return $this->error($exception->getMessage(), 404);
        }
        abort(404);
    }
    
    if ($exception instanceof ValidationException) {
        if ($this->needsJson($request)) {
            return $this->error($exception->errors(), 422);
        }
    }
    
    if ($exception instanceof CustomException) {
        if ($this->needsJson($request)) {
            return $this->error($exception->getMessage(), $exception->getHttpCode());
        }
    }
    
    return parent::render($request, $exception); 
}
```

By utilizing the `render()` function, we can handle different types of
exceptions in a centralized manner, rather than scattering try-catch blocks
throughout our controllers.

In addition to error handling in controllers, let's consider database
transactions, where try-catch blocks are also frequently used:

```php
DB::beginTransaction();

try {
    DB::insert(...);
    DB::insert(...);
    DB::insert(...);
    
    DB::commit();
    // all good
} catch (\Exception $e) {
    DB::rollback();
    // something went wrong
}
```

However, Laravel provides a more elegant solution for database transactions
using the `DB::transaction()` function. This approach eliminates the need for
explicit try-catch blocks and manual calls to `DB::commit()` and
`DB::rollback()`. See the
[Laravel documentation on database transactions](https://laravel.com/docs/database#database-transactions)
for more information.

```php
DB::transaction(function () {
    DB::table('users')->update(['votes' => 1]);
    DB::table('posts')->delete();
});
```

When an error occurs within the closure passed to the `transaction()` function,
Laravel will automatically handle the rollback and invoke the `render()`
function in `App\Exceptions\Handler`, as mentioned earlier.

By adopting these approaches, we can simplify our code, improve maintainability,
and leverage Laravel's built-in error handling capabilities.

In conclusion, it's important to avoid abusing try-catch blocks in Laravel
controllers unless absolutely necessary. Laravel's error handling mechanisms
and the `render()` function in `App\Exceptions\Handler` provide more efficient
and centralized ways to handle exceptions. Additionally, the `DB::transaction()`
function simplifies database transactions by eliminating the need for explicit
try-catch blocks. Embrace these features to enhance your Laravel development
workflow.
