---
layout: post
title: "Runtime Classes. A experiment with PHP and Object Oriented Programming"
date: "2011-08-08"
tags: 
  - "php"
  - "poo"
  - "technology"
  - "experiment"
  - "poo"
  - "tdd"
---

Last week I was thinking about creation of a new type of classes. PHP classes but created dynamically at run time. When this idea was running through my head I read the following [article](http://dhotson.tumblr.com/post/1167021666/php-object-oriented-programming-reinvented) and I wanted to write something similar. Warning: Probably that it is something totally useless, but I wanted to create a working prototype (and it was fun to do it ;) ). Let's start:

We are going to crate something like this:

```php
class Human
{
  private $name;
  function __construct($name)
  {
    $this->name = $name;
  }
 
  function hello()
  {
    return "{$this->name} says hello";
  }
}
 
$gonzalo = new Human('Gonzalo');
$peter = new Human('Peter');
 
echo $gonzalo->hello(); // outputs: Gonzalo says hello
echo $peter->hello(); // outputs: Peter says hello
```

but without using class statement, adding the constructor and hello method dinamically.

```php
function testSimpleUsage()
{
  $human = HClass::define()
    ->fct(HClass::__construct, function($self, $name) {$self->name = $name;})
    ->fct('hello', function($self) {
        return "{$self->name} says hello";
      });
 
  $gonzalo = $human->create('Gonzalo');
  $peter = $human->create('Peter');
 
  $this->assertEquals("Gonzalo says hello", $gonzalo->hello());
  $this->assertEquals("Peter says hello", $peter->hello());
}
```

I also want to test undefinded functions with an exception:

```php
function testCallingUndefinedFunctions()
{
  $human = HClass::define()
    ->fct(HClass::__construct, function($self, $name) {$self->name = $name;})
    ->fct('hello', function($self) {
            return "{$self->name} says hello";
          });
 
  $gonzalo = $human->create('Gonzalo');
  $this->setExpectedException('Exception', "ERROR Method 'goodbye' does not exits");
  $gonzalo->goodbye();
}
```

And simple inheritance too 

```php
function testInheritance()
{
  $human = HClass::define()
    ->fct(HClass::__construct, function($self, $name) {$self->name = $name;})
    ->fct('hello', function($self) {
            return "{$self->name} says hello";
          });
 
  $shyHuman = HClass::define($human)
    ->fct('hello', function($self) {
            return "{$self->name} is shy and don't says hello";
          });
 
  $gonzalo = $human->create('Gonzalo');
  $peter = $shyHuman->create('Peter');
 
  $this->assertEquals("Gonzalo says hello", $gonzalo->hello());
  $this->assertEquals("Peter is shy and don't says hello", $peter->hello());
}
```

Now we are going to create dinamically functions:

```php
function testDinamicallyFunctionCreation()
{
  $human = HClass::define()
    ->fct(HClass::__construct, function($self, $name) {$self->name = $name;})
    ->fct('hello', function($self) {
            return "{$self->name} says hello";
          });
 
  $gonzalo = $human->create('Gonzalo');
  $this->assertEquals("Gonzalo says hello", $gonzalo->hello());
 
  try {
    $gonzalo->goodbye();
  } catch (Exception $e) {
    $this->assertEquals("ERROR Method 'goodbye' does not exits", $e->getMessage());
  }
 
  $human->fct('goodbye', function($self) {
            return "{$self->name} says goodbye";
          });
 
  $this->assertEquals("Gonzalo says goodbye", $gonzalo->goodbye());
}
```

And that's it. It works. probably with PHP5.4 we can drop the "$self" variable. It's an ugly trick to pass the real instance of the class to the callback, thanks to the "Added closure $this support back" feature added in the new version of PHP. But at least now we need to it.

And now another turn the screw. Let's try to create the [FizzBuzz](http://codingdojo.org/cgi-bin/wiki.pl?KataFizzBuzz) kata with those "Runtime Classes". I will create two versions of FizzBuzz. Here you can see my implementation of both versions with "standard" PHP. Whith a [simple](https://github.com/12meses12katas/Marzo-FizzBuzz/tree/master/gonzalo123/php2) class, and [another](https://github.com/12meses12katas/Marzo-FizzBuzz/tree/master/gonzalo123/php) one with Two classes and Dependency Injection. Now using the experiment:

```php
function testFizzBuzz()
{
  $fizzBuzz = HClass::define()
    ->fct('run', function($self, $elems = 100) {
        list($fizz, $buzz) = array('Fizz', 'Buzz');
        return array_map(function ($element) use ($fizz, $buzz) {
            $out = array();
            if ($element % 3 == 0 || strpos((string) $element, '3') !== false ) {
              $out[] = $fizz;
            }
            if ($element % 5 == 0 || strpos((string) $element, '5') !== false ) {
              $out[] = $buzz;
            }
            return (count($out) > 0) ? implode('', $out) : $element;
          }, range(0, $elems-1));
      });
  $fizzBuzz = $fizzBuzz->create();
  $arr = $fizzBuzz->run();
 
  $this->assertEquals(count($arr), 100);
  $this->assertEquals($arr[1],  1);
  $this->assertEquals($arr[3],  'Fizz');
  $this->assertEquals($arr[4],  4);
  $this->assertEquals($arr[5],  'Buzz');
  $this->assertEquals($arr[6],  'Fizz');
  $this->assertEquals($arr[20], 'Buzz');
  $this->assertEquals($arr[13], 'Fizz');
  $this->assertEquals($arr[15], 'FizzBuzz');
  $this->assertEquals($arr[53], 'FizzBuzz');
}
```

```php
function testAnotherFizzBuzzImplementationWithDependencyInjection()
{
  $fizzBuzz = HClass::define();
  $fizzBuzz->fct(HClass::__construct, function($self, $fizzBuzzElement) {
       $self->fizzBuzzElement = $fizzBuzzElement;
     });
  $fizzBuzz->fct('run', function($self, $elems = 100) {
       $out = array();
       foreach (range(1, $elems) as $elem) {
         $out[$elem] =  $self->fizzBuzzElement->render($elem);
       }
       return $out;
     });
   
  $fizzBuzzElement = HClass::define()
    ->fct('render', function($self, $element) {
        list($fizz, $buzz) = array('Fizz', 'Buzz');
        $out = array();
        if ($element % 3 == 0 || strpos((string) $element, '3') !== false ) {
          $out[] = $fizz;
        }
 
        if ($element % 5 == 0 || strpos((string) $element, '5') !== false ) {
          $out[] = $buzz;
        }
        return (count($out) > 0) ? implode('', $out) : $element;
    });
 
  $fbe = $fizzBuzzElement->create();
 
  $this->assertEquals($fbe->render(1),  1);
 
  $this->assertEquals($fbe->render(2),  2);
  $this->assertEquals($fbe->render(3),  'Fizz');
  $this->assertEquals($fbe->render(4),  4);
  $this->assertEquals($fbe->render(5),  'Buzz');
  $this->assertEquals($fbe->render(6),  'Fizz');
  $this->assertEquals($fbe->render(20), 'Buzz');
  $this->assertEquals($fbe->render(13), 'Fizz');
  $this->assertEquals($fbe->render(15), 'FizzBuzz');
  $this->assertEquals($fbe->render(53), 'FizzBuzz');
 
  $fb = $fizzBuzz->create($fbe);
  $arr = $fb->run();
 
  $this->assertEquals(count($arr), 100);
  $this->assertEquals($arr[1],  1);
  $this->assertEquals($arr[3],  'Fizz');
  $this->assertEquals($arr[4],  4);
  $this->assertEquals($arr[5],  'Buzz');
  $this->assertEquals($arr[6],  'Fizz');
  $this->assertEquals($arr[20], 'Buzz');
  $this->assertEquals($arr[13], 'Fizz');
  $this->assertEquals($arr[15], 'FizzBuzz');
  $this->assertEquals($arr[53], 'FizzBuzz');
}
```

And that's all. As I said before probably this "Hybrid Classes" or "Runtime Classes" (I don't know how to name them) are totally useless, but it's fun to do it.

```php
phpunit HclassTest.php 
PHPUnit 3.4.5 by Sebastian Bergmann.
 
......
 
Time: 0 seconds, Memory: 5.00Mb
 
OK (6 tests, 39 assertions)
```

Full code on [github](https://github.com/gonzalo123/HClass)
