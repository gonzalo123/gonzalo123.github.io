---
layout: post
title: "Checking the performance reading arrays with PHP"
date: "2011-08-15"
tags: 
  - "micro-optimizations"
  - "php"
  - "technology"
  - "arrays"
---

Reading last Fabien Potencier's [post](http://fabien.potencier.org/article/48/the-php-ternary-operator-fast-or-not) I see something interesting: "_People like micro-optimizations. They are easy to understand, easy to apply... and useless_". It's true. We don't need to obsess with small micro-optimizations. As someone said in a comment "_developer time is more precious than processor time_". It's true again. But normally our code is coded once and executed thousands of times, and we must keep in mind that CPU time many times means money. We need to balance it (as always). But, WTH. I like micro-optimizations, and here comes another one: Checking the access to arrays with PHP.

```php
$limit = 1000000;
$randomKey = rand(0, $limit);
 
// we create a huge array
$arr = array();
for($i=0; $i<=$limit; $i++) {
    $arr[$i] = $i;
}
 
error_reporting(-1);
$time = microtime(TRUE);
$mem = memory_get_usage();
 
$value = $arr[$randomKey];
for($i=0; $i<=$limit; $i++) {
    echo $value;
}
 
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'microtime' => microtime(TRUE) - $time));

```

with this first example we will loop over a big for statement and we are going to retrieve the value of an array.

```
[memory] => 5.6837158203125 [microtime] => 0.32066202163696
```

Now we're going to do the same, but getting the value of the array inside the loop:

```php
for($i=0; $i<=$limit; $i++) {
    $value = $arr[$randomKey];
    echo $value;
}
```

```
[memory] => 5.6837158203125 [microtime] => 0.35844492912292
```

and now without creatin the extra variable $value

```php
for($i=0; $i<=$limit; $i++) {
    echo $arr[$randomKey];
}
```

```
[memory] => 5.683666229248 [microtime] => 0.35710000991821
```

## Conclusion

As we can see the memory usage is almost the same, and the time is almost the same too. Obviously fetching the value of the array outside the loop is faster than doing inside the loop. But we must take into account that we're working with big arrays. With small ones those times are very similar. This time we don't have any significant outcome. Both methods have the same behaviour. Maybe it can be obvious, but I wanted to check it out.
