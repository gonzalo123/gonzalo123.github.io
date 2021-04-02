---
layout: post
permalink: /:year/:month/:day/:title/
title: "Things I miss in PHP: Function decorators"
date: "2009-12-09"
tags: 
  - "php"
---

## The problem:

I want to build a set of classes accessible via REST interface. One example [here](http://weierophinney.net/matthew/archives/228-Building-RESTful-Services-with-Zend-Framework.html) with ZF. My idea is allow users to access via REST to my set of classes. It's no so difficult. But the problem appears when I want to set an authorization engine to my REST web service. Public functions can be called but sometimes user must be authenticated (with a trusted cookie).

I have my class:

```php
class Lib_Myclass
{
    public function publicFunction($var1)
    {
        return "Hello {$var1}";
    }
 
    public function privateFunction($var1)
    {
        return "private Hello {$var1}";
    }
}

```

publicFunction can be called without login but privateFunction only after login

I have done similar with python and Google App Engine. With python I use decorators and its very clean and easy

```php
class Myclass:
    def publicFunction(self, var1):
        return "Hello %s" % var1
 
    @private
    def privateFunction(self, var1):
        return "private Hello %s" % var1
```

And I define my decorator 'private':

```php
def private(funcion):
    def _private(*list_args):
        if isValidUser():
            valor = funcion(*list_args)
        else:
            value = ('You must be logged')
        return value
    return _private
```

Really simple and clean. But in PHP I don't know how to do it in a simple way. I don't want to do reflections in every call and check if the function is allowed or not.

## My solution:

Instead of having one class with public and private functions I divide it in two classes: one for the public function and another one for the private (private here means logged sessions not private keyword in PHP). I also create two empty Interfaces PublicAccess and PrivateAccess and using PHP's instanceof I can throw an exception if user without login tries to call any function in a class with the interface PrivateAccess

```php
interface PublicAccess{}
interface PrivateAccess{}
 
class Lib_Myclass1 implements PublicAccess
{
    public function publicFunction($var1)
    {
        return "Hello {$var1}";
    }
}
 
class Lib_Myclass2 implements PrivateAccess
{
 
    public function privateFunction($var1)
    {
        return "private Hello {$var1}";
    }
}
```

I prefer the python's solution
