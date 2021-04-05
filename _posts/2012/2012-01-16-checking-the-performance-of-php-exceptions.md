---
layout: post
title: "Checking the performance of PHP exceptions"
date: "2012-01-16"
tags: 
  - "micro-optimizations"
  - "php"
  - "technology"
  - "benchmark-test"
  - "exceptions"
  - "language-php"
  - "performance"
---

Sometimes we use exceptions to manage the flow of our scripts. I imagine that the use of exceptions must have a performance lack. Because of that I will perform a small benchmark to test the performance of one simple script throwing exceptions and without them. Let’s start:

First a silly script to find even numbers (please don't use it it's only for the benchmanrk :) ) 

```php
error_reporting(-1);
$time = microtime(TRUE);
$mem = memory_get_usage();
 
$even = $odd = array();
foreach (range(1, 100000) as $i) {
  try {
    if ($i % 2 == 0) {
      throw new Exception("even number");
    } else {
      $odd[] = $i;
    }
  } catch (Exception $e) {
    $even[] = $i;
  }
}
echo "odds: " . count($odd) . ", evens " . count($even);
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'microtime' => microtime(TRUE) - $time));
```

And now the same script without exceptions.

```php
error_reporting(-1);
$time = microtime(TRUE);
$mem = memory_get_usage();
 
$even = $odd = array();
foreach (range(1, 100000) as $i) {
    if ($i % 2 == 0) {
        $even[] = $i;
    } else {
        $odd[] = $i;
    }
}
 
echo "odd: " . count($odd) . ", evens " . count($even);
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'microtime' => microtime(TRUE) - $time));
```

The outcomes: **with exceptions** **memory**: 10.420181274414 **microtime**: 1.1479668617249 0.19249302864075 (without xdebug)

**without exceptions** **memory**: 10.418941497803 **microtime**: 0.14752888679505 0.1234929561615

As we can see the use of memory is almost the same and ten times faster without exceptions.

I have done this test using a VM box with 512MB of memory and PHP 5.3. Now we are going to do the same test with a similar host. The same configuration but PHP 5.4 instead of 5.3

**PHP 5.4:** **with exceptions** **memory**: 7.367259979248 **microtime**: 0.1864490332

**without exceptions** **memory**: 7.3669052124023 **microtime**: 0.089046955108643

I'm really impressed with the outcomes. The use of memory here with PHP 5.4 is much better now and the execution time better too (ten times faster).

According to the outcomes my conclusion is that the use of exceptions in the flow of our scripts is not as bad as I supposed. OK in this example the use of exceptions is not a good idea, but in another complex script exceptions are really useful. We also must take into account the tests are iterations over 100000 to maximize the differences. So I will keep on using exceptions. This kind of micro-optimization not seems to be really useful. What do you think?
