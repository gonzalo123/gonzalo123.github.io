---
layout: post
title: "Building a BDD framework with PHP"
date: "2013-08-19"
tags: 
  - "php"
  - "phpunit"
  - "symfony"
  - "technology"
  - "bdd"
  - "jasmine"
  - "language-php"
  - "tdd"
---

Do you know [Jasmine](http://pivotal.github.io/jasmine/), the BDD framework for JavaScript? This holidays I was looking for something like that in PHP and I didn't found anything similar (please let me know if I'm wrong) (now I know a few of them thanks to [Dave's](https://twitter.com/davedevelopment) comment) . Because of that, and as an exercise, I hack a little bit building something similar for PHP.

I want to write as less code as I can (it's only a proof of concept), so I will reuse the assertion framework or PHPUnit. As I've seen when studying [Behat](http://behat.org/), we can use the assertion part as standalone functions. We only need to include _vendor/phpunit/phpunit/PHPUnit/Framework/Assert/Functions.php_ file

Here you can see one example. 

```php
class StringCalculator
{
    public function add($string)
    {
        return (int)array_sum(explode(",", $string));
    }
}
 
$stringCalculator = new StringCalculator;
 
describe("add mull returns zero", function () use ($stringCalculator) {
        assertEquals(null, $stringCalculator->add(""));
    });
 
describe("1,1 should return 2", function () use ($stringCalculator) {
        assertEquals(2, $stringCalculator->add("1,1"));
    });
```

We also can use something similar than DataProvider in PHPUnit:

```javascript
describe("add number returns number", function ($expected, $actual, $message) use ($stringCalculator) {
        assertEquals($expected, $stringCalculator->add($actual), $message);
    }, [
        ['expected' => 1, 'actual' => "1", 'message' => 'add 1'],
        ['expected' => 2, 'actual' => "2", 'message' => 'add 1'],
        ['expected' => 10, 'actual' => "10", 'message' => 'add 10'],
    ]);  
```

And if we need mocks we can use Mockery, for example:

```php
class Temperature
{
 
    public function __construct($service)
    {
        $this->_service = $service;
    }
 
    public function average()
    {
        $total = 0;
        for ($i=0;$i<3;$i++) {
            $total += $this->_service->readTemp();
        }
        return $total/3;
    }
}
 
$service = m::mock('service');
 
describe("testing mocks with mockery", function() use ($service) {
        $service->shouldReceive('readTemp')->andReturn(11, 12, 13);
        $temperature = new Temperature($service);
        assertEquals(12, $temperature->average(), "dummy message");
    });
```

I've created an small console application to run the test suites using symfony/console and symfony/finder components. We can run it with:

```commandline
php ./bin/console.php texter:run ./tests
```

Beware because this library is a proof of concept. There're a lot of remaining things. What do you think?

Source code at [github](https://github.com/gonzalo123/texter)
