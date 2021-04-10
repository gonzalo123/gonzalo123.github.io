---
layout: post
title: "Reflection over PHPDoc with PHP"
date: "2011-04-04"
tags: 
  - "php"
  - "tips"
  - "phpdoc"
  - "reflection"
---

I want to parse PHPDoc code. Let me explain a little bit what I want to do. Imagine a dummy function documented with PHPDoc:

```php
class Foo
{
    /**
     * @param String $param1
     * @param String $param2
     * @return String
     */
    public function dummy($param1, $param2)
    {
        return "Hello World";
    }
}
```

PHP has a great reflection [API](http://php.net/manual/en/class.reflection.php), but as at least in the current PHP version (as far as I know) we only can get the PHPDoc as a string, without parse it. I need to get the parameters and the type of them with reflection. Parameters are the easy part:

```php
$class = new ReflectionClass($obj);
$reflect = $class->getMethod($function);
foreach ($reflect->getParameters() as $i => $param) {
}
```

But the type is different. PHP is a loosely/weak typed, dynamic language, because of that we need to parse the PHPDoc string to get the “real” type of the variables. I put “real” quoted because if we set a variable as a String within PHPDoc, we can use an Integer at runtime. We can cast variables but only with complex types (classes), and not with primary types as String or Integer. I’ve read it will be available on PHP's next releases, but now we need to use exotic tricks (as the trick I will show now).

I’m not a master in regular expressions (in fact I hate them), so it looks like a hard work for me, but I realized that Zend Framework does something similar when it builds a [XMLRPC server](http://framework.zend.com/manual/en/zend.xmlrpc.server.html). I don’t use ZF in this project, and I don’t want to include the whole library only for parsing the PHPDoc. But there’s a great thing. ZF is an open source library, really well coded and documented. Because of that I dive into the ZF code, looking for the functionality that I need and adapt it.

```php
function processPHPDoc(ReflectionMethod $reflect)
{
    $phpDoc = array('params' => array(), 'return' => null);
    $docComment = $reflect->getDocComment();
    if (trim($docComment) == '') {
        return null;
    }
    $docComment = preg_replace('#[ \t]*(?:\/\*\*|\*\/|\*)?[ ]{0,1}(.*)?#', '$1', $docComment);
    $docComment = ltrim($docComment, "\r\n");
    $parsedDocComment = $docComment;
    $lineNumber = $firstBlandLineEncountered = 0;
    while (($newlinePos = strpos($parsedDocComment, "\n")) !== false) {
        $lineNumber++;
        $line = substr($parsedDocComment, 0, $newlinePos);
 
        $matches = array();
        if ((strpos($line, '@') === 0) && (preg_match('#^(@\w+.*?)(\n)(?:@|\r?\n|$)#s', $parsedDocComment, $matches))) {
            $tagDocblockLine = $matches[1];
            $matches2 = array();
 
            if (!preg_match('#^@(\w+)(\s|$)#', $tagDocblockLine, $matches2)) {
                break;
            }
            $matches3 = array();
            if (!preg_match('#^@(\w+)\s+([\w|\\\]+)(?:\s+(\$\S+))?(?:\s+(.*))?#s', $tagDocblockLine, $matches3)) {
                break;
            }
            if ($matches3[1] != 'param') {
                if (strtolower($matches3[1]) == 'return') {
                    $phpDoc['return'] = array('type' => $matches3[2]);
                }
            } else {
                $phpDoc['params'][] = array('name' => $matches3[3], 'type' => $matches3[2]);
            }
 
            $parsedDocComment = str_replace($matches[1] . $matches[2], '', $parsedDocComment);
        }
    }
    return $phpDoc;
}
```

Our processPHPDoc function takes as argument $reflect (ReflectionMethod) and returns an array

```
Array
(
    [params] => Array
        (
            [0] => Array
                (
                    [name] => $param1
                    [type] => String
                )
 
            [1] => Array
                (
                    [name] => $param2
                    [type] => String
                )
        )
 
    [return] => Array
        (
            [type] => String
        )
)
```

And that was exactly what I needed. Open Source is great.
