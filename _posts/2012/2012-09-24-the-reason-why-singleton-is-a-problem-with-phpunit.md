---
layout: post
title: "The reason why singleton is a \"problem\" with PHPUnit"
date: "2012-09-24"
tags: 
  - "php"
  - "phpunit"
  - "tdd"
  - "technology"
  - "tdd"
---

[Singleton](http://en.wikipedia.org/wiki/Singleton_pattern) is a common design pattern. It's easy to understand and we can easily implement it with PHP:

```php
<?php
class Foo
{
    static private $instance = NULL;
 
    static public function getInstance()
    {
        if (self::$instance == NULL) {
            self::$instance = new static();
        }
        return self::$instance;
    }
}
```

Maybe this pattern is not as useful as it is in J2EE world. With PHP everything dies within each request, so we cannot persist our instances between requests (without any persistent mechanism such as databases, memcached or external servers). But at least in PHP we can share the same instance, with this pattern, in our script. No matter were we are, we use _Foo::getInstance()_ and we get our instance. It useful when we work, for example, with database connections and service containers.

If we work with TDD we most probably use [PHPUnit](http://www.phpunit.de/manual/3.7/en/index.html). Imagine a simple script to test the instantiation of our Foo class 

```php
class FooTest extends PHPUnit_Framework_TestCase
{
    public function testGetInstance()
    {
        $this->assertInstanceOf('Foo', Foo::getInstance());
    }
}
```

All is green. It works.

Imagine now we want to "improve" our Foo class and we want to create a counter:

```php
public function testCounter()
{
    $foo = Foo::getInstance();
    $this->assertEquals(0, $foo->getCounter());
    $foo->incrementCounter();
    $this->assertEquals(1, $foo->getCounter());
}
```

Now our test fails, So we change our implementation of the class to pass the test and we add to Foo class

```php
private $counter = 0;
 
public function getCounter()
{
    return $this->counter;
}
 
public function incrementCounter()
{
    $this->counter++;
}
```

It's green again. We are using a singleton pattern to get the instance of the class and we are using TDD, so what's the problem? I will show you with an example. Imagine we need to improve our Foo class and we want a getter to say if counter is even or not. We add a new test:

```php
public function testIsOdd()
{
    $foo = Foo::getInstance();
    $this->assertEquals(FALSE, $foo->counterIsOdd(), '0 is even');
    $foo->incrementCounter();
    $this->assertEquals(TRUE, $foo->counterIsOdd(), '1 is uneven');
}
```

And we implement the code to pass the test again:

```php
public function counterIsOdd()
{
    return $this->counter % 2 == 0 ? FALSE : TRUE;
}
```

But it doesn't work. Our test is red now. Why? The answer is simple. We are sharing the same instance of Foo class in both tests, so in the first one our counter start with 0 (the default value) but with the second one it start with 1, because the first test has changed the state of the instance (and we are reusing it).

This is a trivial example, but real world problems with this issue are a bit nightmare. Imagine, for example, that we use singleton pattern to get the database connection (a common usage of this pattern). Suddenly we break one test (because we have created a wrong SQL statement). Imagine we have a test suite with 100 assertions. We have broken one assertion (the second one, for example), and our PHPUnit script will say that we have 99 errors (the second one and all the following ones). That happens because our script can't reuse the database connection (it's broken in the second test). Too much garbage in the output of the PHPUnit script and that's not agile.

How to solve it? We have two possible solutions. The first one is avoid singletons as a plague. We have alternatives such as dependency injection, but if we really want to use them (they are not forbidden, indeed) we need to remember to write one static function in our class to reset all our static states and call after each test. PHPUnit allow to execute something at the end of each test with tearDown() function. So we add to our test:

```php
class FooTest extends PHPUnit_Framework_TestCase
{
...
    public function tearDown()
    {
        Foo::tearDown();
    }
}
```

And now we implement the tearDown function in our Foo Class 

```php
public static function tearDown()
{
    static::$instance = NULL;
}
```

And all works again. All is green and green is cool, isn't it? :)

_Note: Take into account that "problem" is between quotes in the title. That's beacause singleton is not a "real" problem. We can use it, it isn't forbiden, but we need to realize the collateral effects that its usage can throw to our code._
