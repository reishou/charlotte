---
layout: post
title: "#1 - Laravel: Optimizing Likeable"
slug: laravel-optimizing-likable
date: 2020-05-21T09:15:00+0700
categories: [laravel]
image: likeable.jpg
tags: [
    'backend',
    'laravel',
    'likable',
    'optimize',
    'eloquent'
]
---
The "likeable" feature in Laravel has become a staple for many developers.
You've probably searched the internet for ways to implement it, but in this
blog post, we'll take it a step further. We'll explore the common challenges
faced when using the likeable feature and unveil some powerful optimizations.

Imagine you're implementing the like feature for players. Typically, when
fetching a list of players, you need to determine whether the currently
logged-in user has liked a particular player. Let's try a clever approach
by leveraging Eloquent's accessor capabilities.

```php
public function getIsLikedAttribute()
{
    $liked = $this->likes()->where( 'user_id', auth()->id())->first();
    return (bool)$liked;
}
```

With this implementation, the returned data will include an `is_liked` field
that accurately reflects whether the logged-in user has liked each player or
not.

```json
{
  "players": [
    {
      "id": 89573795441935385,
      "name": "Alex",
      "created_at": "2019-10-11T12:07:08+0000",
      "updated_at": "2020-05-17T14:00:12+0000",
      "is_liked": true
    },
    {
      "id": 89573795265774616,
      "name": "Alex",
      "created_at": "2019-10-11T12:07:08+0000",
      "updated_at": "2020-05-17T14:00:12+0000",
      "is_liked": false
    }
  ]
}
```

At first glance, this approach may seem simple and straightforward. However,
it presents a performance challenge when dealing with large datasets or
paginated results limited to 25 items.

```json
{
  "database": {
    "total": 29,
    "items": [
      {
        "connection": "mysql",
        "query": "select * from `likes` where `likes`.`likeable_id` = `194456710046286896` and `likes`.`likeable_id` is not null and `likes`.`likeable_tyPe` = `App\\Player` and `user_id` = `157426748995143158` and `likes`.`deleted_at` is null limit 1;",
        "time": 0.38
      },
      {
        "connection": "mysql",
        "query": "select * from `likes` where `likes`.`likeable_id` = `194456710021121071` and `likes`.`likeable_id. is not null and `likes`.`likeable_tyPe` = `App\\Player` and `user_id` = `157426748995143158` and `likes`.`deleted_at` is null limit 1;",
        "time": 0.29
      }
    ]
  }
}
```

The culprit lies within the `getIsLikedAttribute()` method, where we
unintentionally fall into the notorious N+1 problem, executing an
additional query for each player. But fear not! We have an exciting
optimization technique to eliminate these performance bottlenecks.

```php
public function liked()
{
    return $this->morphOne(Like::class, 'likeable')
        ->where('user_id', auth()->id())
        ->select(rid', 'created_at', 'updated_at1);
}
```

Instead of executing queries within the accessor, we'll leverage a
powerful Eloquent relationship called liked. By utilizing the `with()`
method, we can eagerly load the liked relationship and transform the implementation.

```json
{
  "players": [
    {
      "id": 194456710046286896,
      "name": "Alex",
      "created_at": "2020-03-04T05:10:46+0000",
      "updated_at": "2020-05-17T14:00:12+0000",
      "liked": {
        "id": 241465878896444422,
        "likeable_id": 194456710046286896,
        "created_at": "2020-05-08T01:49:35+0000",
        "updated_at": "2020-05-21T04:12:15+0000"
      }
    },
    {
      "id": 194456710021121071,
      "name": "Tom",
      "created_at": "2020-03-04T05:10:46+0000",
      "updated_at": "2020-05-17T14:00:12+0000",
      "liked": null
    }
  ]
}
```

Voilà! The `is_liked` field has now become `liked`. It no longer represents a simple
boolean value but instead provides an object or null depending on the user's like
status. As a result, the number of queries dramatically decreases, and your
application's performance receives a significant boost.

```json
{
  "database": {
    "total": 4,
    "items": [
      {
        "connection": "mysql",
        "query": "select count(*) as aggregate from `players` where `players`.`deleted_at` is null;",
        "time": 2.55
      },
      {
        "connection": "mysql",
        "query": "select `id`, `sync_player_id`, `name`, `localized_name`, `gender`, `date_of_birth`, `registration_number`, `class`, `registered_date`, `nationality`, `prefecture_id`, `graduate`, `current_rank_code`, `next_rank_code`, `note`, `created_at`, `updated_at` from `players` where `players`.`deleted_at` is null order by `updated_at` desc, `id` desc limit 25 offset 0;",
        "time": 0.59
      },
      {
        "connection": "mysql",
        "query": "select `id`, `player_id`, `type, `expiration_date`, `deleted_date., `registered_date`, `updated_date` from `player_licenses` where `player_licenses`.`player_id` in (89573795265774616, 89573795441935385, 89573795735536666, 89573795911697435, 89573796079469596, 89573796222075933, 89573796398236702, 89573796574397471, 89573796725392416, 89573796876387361, 89573797044159522, 89573797220320291, 89573797413258276, 89573797606196261, 89573797773968422, 89573797924963367, 89573798075958312, 89573798243730473, 96879317530706986, 194456709912069163, 194456709937234988, 194456709970789421, 194456709995955246, 194456710021121071, 194456710046286896);",
        "time": 0.71
      },
      {
        "connection": "mysql",
        "query": "select `id`, `likeable_id`, `created_at`, `updated_at` from `likes` where `user_id` = `157426748995143158` and `likes`.`likeable_id` in (89573795265774616, 89573795441935385, 89573795735536666, 89573795911697435, 89573796079469596, 89573796222075933, 89573796398236702, 89573796574397471, 89573796725392416, 89573796876387361, 89573797044159522, 89573797220320291, 89573797413258276, 89573797606196261, 89573797773968422, 89573797924963367, 89573798075958312, 89573798243730473, 96879317530706986, 194456709912069163, 194456709937234988, 194456709970789421, 194456709995955246, 194456710021121071, 194456710046286896) and `likes`.`likeable_type` = `App\\Player` and `likes`.`deleted_at` is null;",
        "time": 0.88
      }
    ]
  }
}
```

Optimizing the likeable feature in Laravel is not only about functionality but
also performance. By employing the power of relationships and smart data
retrieval techniques, you can supercharge your application's likeability and
provide a seamless user experience.

So, let's dive into the world of Laravel optimizations and unlock the true
potential of your likeable feature!
