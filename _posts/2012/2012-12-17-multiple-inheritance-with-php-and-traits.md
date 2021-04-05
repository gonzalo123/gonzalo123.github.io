---
layout: post
title: "Multiple inheritance with PHP and Traits"
date: "2012-12-17"
tags: 
  - "php"
  - "technology"
  - "multiple-inheritance"
  - "traits"
---

Multiple inheritance isn't allowed in PHP. With Python we can do things like that:

```python
class ClassName(Base1, Base2):
    ....
```

That's no possible with PHP (in Java is not possible either), but today we can do something similar (is not the exactly the same) with **Traits**. Let me explain that: Instead of classes we can create Traits:

```php
<?php
trait Base1
{
    public function hello1($name)
    {
        return "Hello1 {$name}";
    }
} 
 
trait Base2
{
    public function hello2($name)
    {
        return "Hello2 {$name}";
    }
} 
```

And now we can use those traits (instead of to extend multiple classes)

```php
<?php
class ClassName
{
    use Base1, Base2;
} 
```

The main reason because multiple inheritance isn't allowed in PHP, Java and another languages is is due to the collisions. If we extends from two classes with the same method, which is the good one? Python solves this problem with a easy solution: The first one is the good one:

```python
class Base1:
    def hello1(self, name):
        return "Hello1 " + name
 
class Base2:
    def hello1(self, name):
        return "Hello2 " + name
 
class ClassName(Base1, Base2):
    pass
c = ClassName()
print c.hello1("Gonzalo")
```

Will output "Hello1 Gonzalo" but if change the inheritance order to:

```python
class ClassName(Base2, Base1):
    pass
```

The output will be "Hello2 Gonzalo"

Traits in PHP doesn't solve the problem "out of the box". If we use the following script:

```php
<?php
trait Base1
{
    public function hello1($name)
    {
        return "Hello1 {$name}";
    }
}
 
trait Base2
{
    public function hello1($name)
    {
        return "Hello2 {$name}";
    }
}
 
class ClassName
{
    use Base1, Base2;
}
 
$class = new ClassName();
echo $class->hello1("Gonzalo");
```

Our script will throw a Fatal error:

_PHP Fatal error: Trait method hello1 has not been applied, because there are collisions with other trait methods on ClassName on line 23_

If we want to avoid collisions, we need to describe explicitly what function we will use. It's not difficult:

```php
<?php
trait Base1
{
    public function hello1($name)
    {
        return "Hello1 {$name}";
    }
}
 
trait Base2
{
    public function hello1($name)
    {
        return "Hello2 {$name}";
    }
}
 
class ClassName
{
    use Base1, Base2 {
        Base1::hello1 insteadof Base2;
    }
}
 
$class = new ClassName();
echo $class->hello1("Gonzalo");
```
