---
layout: post
title: "New features in PHP5.4 alpha1"
date: "2011-07-04"
tags: 
  - "php"
  - "technology"
---

We already have the php5.4 alpha ready for testing. This new release has a few features that I really like. The lack of those features gave me problems in the past. Because of that, I'm very excited waiting for the new stable release of PHP 5.4. Now I'm going to list the new features I like. There're more features (check the full list [here](http://www.php.net/releases/NEWS_5_4_0_alpha1.txt)) but those ones are the ones I'm really waiting for:

## Added array dereferencing support

When we're doing OO we can do things like that:

```php
class A
{
    public function foo()
    {
        return new B();
    }
}
 
class B
{
    public function bar()
    {
        return "Hi";
    }
}
 
$a = new A();
echo $a->foo()->bar();
```

But if we output an array instead of an instance of class, we will need an extra variable to use it:

```php
class A
{
    public function foo()
    {
        return array('name' => 'Gonzalo');
    }
}
 
$a = new A();
$foo = $a->foo();
echo $foo['name'];
```

Now with PHP 5.4 we'll use:

```php
$a = new A();
echo $a->foo()['name'];
```

Cool, isn't it?

## Added indirect method call through array

Now an array('class', 'method') can be used to call a function. is\_callable will return true.

That means we can do:

```php
class Foo {
   public function bar($name) {
      echo "Hi, $name";
   }
}
 
$f = array('Foo','bar');
echo $f('Gonzalo'); // Hi, Gonzalo
```

More information [here](https://wiki.php.net/rfc/indirect-method-call-by-array-var)

## Added closure $this support back

When we use closures in PHP5.3 and we want to use $this statement, we need to do some strange hacks to use it in the context of the callback. Things like _$that = $this;_ and _use($that)_. It works, but looks like an ugly hack. Now we can do:

```php
class A {
  private $value = 1;
  function single_getter($name) {
    return function() use ($name) {
      return $this->$name;
    };
  }
}
class B {
  private $value = 2;
  function test() {  
    $a = new A;
    $single_getter = $a->single_getter('value');
    print $single_getter();
  }
}  
 
$b = new B;
$b->test();
```

PHP5.3 -> Fatal error: Using $this when not in object context PHP5.4 -> the script will output "1"

More information [here](http://drupal4hu.com/node/291).

## Added support for Traits

And finally we will enjoy of traits: Here we can read a really good [article](http://simas.posterous.com/new-to-php-54-traits) about it:

With Traits we can share interfaces and avoid inheritance nightmares to reuse code. Clean and simple.

## Conclusions

In my handle opinion those new features will help PHP to grow a little bit more and adapt itself to new years. It's a pity we still don't have a short way to define arrays and we still need to use array(1, 2, 3) instead of doing things like \[1, 2, 3\], but I hope this new feature will be available in future versions I'm not sure but I think it's on the roadmap.
