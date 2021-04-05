---
layout: post
title: "How to use eval() without using eval() in PHP"
date: "2012-03-12"
tags: 
  - "php"
  - "technology"
---

Yes I know. [Eval()](http://php.net/manual/en/function.eval.php) is evil. If our answer is to use eval() function, we are probably asking the wrong question. When we see an eval() function all our coding smell's red lights start flashing inside our mind. Definitely it's a bad practice.

But last week I was thinking about it. How can I eval raw PHP code without using the eval function, and I will show you my outcomes.

Imagine this simple script 

```php
<?php
error_reporting(-1);
 
class Foo
{
    private $name;
    public function __construct($name)
    {
        $this->name = $name;
    }
 
    public function hello()
    {
        return $this->name;
    }
}
 
$serializedOutput = null;
foreach (range(1, 100000) as $i) {
    $object = new Foo("name" . $i);
    $out[] = $object->hello();
}
 
$serializedOutput = serialize($out);
 
echo strlen($serializedOutput);
```

Now we are going to the same using eval function. Imagine for example that we are using an online PHP interpreter (yes, it's hard to find examples to use eval()).

```php
<?php
error_reporting(-1);
 
// Our PHP code inside a variable
$phpCode = '
class Foo
{
    private $name;
    public function __construct($name)
    {
        $this->name = $name;
    }
 
    public function hello()
    {
        return $this->name;
    }
}
 
$serializedOutput = null;
foreach (range(1, 100000) as $i) {
    $object = new Foo("name" . $i);
    $out[] = $object->hello();
}
$serializedOutput = serialize($out);
';
// end of variable
eval($phpCode);
 
echo strlen($serializedOutput);
```

Our ugly "eval" version does the same than the original one. Even the performance is almost the same. More or less the same memory usage and speed. The challenge now is to do the same but without using eval().

My idea is simple. Create a temporary file with the PHP source code, include this file with the standard PHP's include functions and destroy the temporary file. Here is the code snippet:

```php
<?php
error_reporting(-1);
 
// Our PHP code inside a variable
$phpCode = '
class Foo
{
    private $name;
    public function __construct($name)
    {
        $this->name = $name;
    }
 
    public function hello()
    {
        return $this->name;
    }
}
 
$serializedOutput = null;
foreach (range(1, 100000) as $i) {
    $object = new Foo("name" . $i);
    $out[] = $object->hello();
}
$serializedOutput = serialize($out);
';
// end of variable
function fakeEval($phpCode) {
    $tmpfname = tempnam("/tmp", "fakeEval");
    $handle = fopen($tmpfname, "w+");
    fwrite($handle, "<?php\n" . $phpCode);
    fclose($handle);
    include $tmpfname;
    unlink($tmpfname);
    return get_defined_vars();
}
extract(fakeEval($phpCode));
echo strlen($serializedOutput);
```

This fake-eval have similar performance than eval version. Maybe a bit worse, probably because of the creation of the temp file, but almost inappreciable.

Maybe is not very useful script and, of course I strongly recommend the first version (without eval and fake-eval) but, what do you think?
