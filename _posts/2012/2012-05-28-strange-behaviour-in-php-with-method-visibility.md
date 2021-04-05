---
layout: post
title: "Strange behavior in PHP with method visibility"
date: "2012-05-28"
tags: 
  - "opinion"
  - "php"
  - "technology"
  - "language-php"
  - "oo"
  - "programming"
---

Normally I feel very comfortable with PHP, but not all is good. There's some things I don’t like. One is the lack of real annotations and another one is this rare behaviour with visibility within the OO. Let me explain this a little bit.

Last week I was refactoring one old script. I removed a coupling problem with DI. Something like this:

```php
class AnotherClass
{
    protected function foo()
    {
        return "bar";
    }
}
 
class OneClass extends AnotherClass{
    private $object;
 
    public function __construct(AnotherClass $object)
    {
        $this->object = $object;
    }
 
    public function myFunction()
    {
        return $this->object->foo();
    }
}
 
$anotherClass = new AnotherClass();
$oneClass = new OneClass($anotherClass);
 
echo $oneClass->myFunction();
```

It works, but I realized that I didn’t need to extend OneClass with AnotherClass (due to the DI), so I removed it. Then the script crashed: Fatal error: Call to protected method AnotherClass::foo() from context 'OneClass'

Obviously it was due to the protected function AnotherClass::foo. But, Why it worked when I extends OneClass with AnotherClass? The visibility is the same.

I reported this “[bug](https://bugs.php.net/bug.php?id=62045)” to the PHP community. PHP community is great. I had an answer very quick. It was not a bug. I needed to read several times the answer to understand it but finally did it.

As someone answer me:

> foo() is protected and was defined in the context of OneClass. The access is done in the context of AnotherClass. AnotherClass is a subclass of OneClass (the context where foo() was defined). Therefore access is granted

In PHP the visibility belongs to the class not to the instance of the class. I understand the reason, but my mind compute it as a bug :( and it isn’t. It's a feature.

What do you think?
