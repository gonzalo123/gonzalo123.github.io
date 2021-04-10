---
layout: post
title: "Playing with the new PHP5.4 features"
date: "2011-11-28"
tags: 
  - "php"
  - "technology"
---

PHP5.4 it's close and it's time to start playing with the new cool features. I've created a new Virtual Machine to play with the new features available within PHP5.4. I wrote a [post](http://gonzalo123.wordpress.com/2011/07/04/new-features-in-php5-4-alpha1/ "New features in PHP5.4 alpha1") with the most exciting features (at least for me) when I saw the feature list in the alpha version. Now the Release Candidate is with us, so it's the time of start playing with them. I also discover really cool features that I pass over in my first review

Let's start:

**Class member access on instantiation.**

Cool! 

```php
class Human
{
    function __construct($name)
    {
        $this->name = $name;
    }
 
    public function hello()
    {
        return "Hi " . $this->name;
    }
}
 
// old style
$human = new Human("Gonzalo");
echo $human->hello();
 
// new cool style
echo (new Human("Gonzalo"))->hello();
```

**Short array syntax** Yeah! 

```php
$a = [1, 2, 3];
print_r($a);
```

**Support for Class::{expr}() syntax** 

```php
foreach ([new Human("Gonzalo"), new Human("Peter")] as $human) {
    echo $human->{'hello'}();
}
```

**Indirect method call through array** 

```php
$f = [new Human("Gonzalo"), 'hello'];
echo $f();
```

**Callable typehint** \

```php
function hi(callable $f) {
    $f();
}
 
hi([new Human("Gonzalo"), 'hello']);
```

**Traits** 

```php
trait FlyMutant {
    public function fly() {
        return 'I can fly!';
    }
}
 
class Mutant extends Human {
    use FlyMutant;
}
 
$mutant = new Mutant("Storm");
echo $mutant->fly();
```

**Array dereferencing support** 

```php
function data() {
    return ['name' => 'Gonzalo', 'surname' => 'Ayuso'];
}
 
echo data()['name'];
```

IDEs don't support (yet) those features. That's means IDEs will mark those new features as syntax errors but I hope that they will support them soon. More info [here](http://php.webtutor.pl/en/2011/09/27/whats-new-in-php-5-4-a-huge-list-of-major-changes/)
