---
layout: post
title: "PHP Redis cache wrapper"
date: "2017-04-19"
categories: 
  - "php"
  - "technology"
tags: 
  - "predis"
  - "redis"
---

Today I want to share a simple Redis Wrapper for PHP I'm using in one project. Nowadays I don't like reinvent the wheel as years ago. I'm not going to create a new Redis Library for PHP there're too many. I normally use [Predis](http://predis/predis) (it's the "de facto" standard, indeed).

As I said before I don't like to re-invent the wheel, but I like to create wrappers to adapt libraries to my common operations. For example with Redis I normally use a lot read key from cache (and create the key in the cache if it doesn't exits). Because of that I've created this simple wrapper to enclose this common operation for me.

The idea is to use something like this: 

```php
use G\Redis\Cache;
use Predis\Client;
 
$cache = new Cache(new Client(json_decode(file_get_contents(__DIR__ . '/conf.json'), true)));
 
$value = $cache-&gt;get("aaa.aaa.aaa", function () {
return [
'a' =&gt; 1,
];
});
 
$cache-&gt;delete("aaa.aaa.aaa");
```

It's very simple. I create a new instance (injecting a Predis instance), and when I want to get the key from Redis I also pass a callback with all that I need if to create the key in the cache if it doesn't exits.

The wrapper's code is very simple also

```php
<?php
namespace G\Redis;

use Predis\ClientInterface;

class Cache
{
    private $redis;

    public function __construct(ClientInterface $redis)
    {
        $this->redis = $redis;
    }

    public function get($key, callable $callback = null, $ttl = null)
    {
        if ($this->redis->exists($key)) {
            return json_decode($this->redis->get($key), true);
        }

        if (!is_callable($callback)) {
            throw new Exception("Key value '{$key}' not in cache and not available callback to populate cache");
        }

        $out = call_user_func($callback);
        if (!is_null($ttl)) {
            $this->redis->set($key, json_encode($out), 'EX', $ttl);
        } else {
            $this->redis->set($key, json_encode($out));
        }

        return $out;
    }

    public function delete($key)
    {
        $keys = $this->redis->keys($key);
        if (count($keys) > 0) {
            $this->redis->del($keys);
        }

        return $keys;
    }
}
```

The code is available in my [github](https://github.com/gonzalo123/redisphpwrapper/) and also in [Packagist](https://packagist.org/packages/gonzalo123/redisphpwrapper)
