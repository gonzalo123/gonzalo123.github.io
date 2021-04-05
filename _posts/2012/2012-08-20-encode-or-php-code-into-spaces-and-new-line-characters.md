---
layout: post
title: "Encode our PHP code into spaces and new line characters"
date: "2012-08-20"
tags: 
  - "php"
  - "ascii-code"
  - "language-php"
---

According to one my last [blog](http://gonzalo123.wordpress.com/2012/03/12/how-to-use-eval-without-using-eval-in-php/ "How to use eval() without using eval() in PHP") post with an exotic usage of dynamic includes with PHP we will keep on with something still more exotic. The idea is to create clean code. And what is cleaner than the blank?

We will encode our PHP code into spaces and new line characters (\\n). The idea is simple. We will translate each character of our script into (n) spaces where (n) is the hexadecimal representation of it's ASCII code.

```php
function decodeText($encodedText) {
    $out = array();
    foreach (explode("\n", $encodedText) as $line) {
        $lineOut = array();
        foreach (explode("\t", $line) as $item) {
            $lineOut[] = chr(strlen($item));
        }
        $out[] = implode(null, $lineOut);
    }
    return implode("\n", $out);
}
 
function encodeText($text) {
    $out = array();
    foreach (explode("\n", $text) as $line) {
        $lineOut = array();
        foreach (str_split($line) as $item) {
            $lineOut[] = str_repeat(" ", ord($item));
        }
        $out[] = implode("\t", $lineOut);
    }
    return implode("\n", $out);
}
```

And now we will create two functions to include our files and parse them with PHP. One function to include from raw input and another to include files.

```php
function includeFromFile($file)
{
    $raw = file_get_contents($file);
    includeFromRaw($raw);
}
 
function includeFromRaw($rawText)
{
    $decodedCode = decodeText($rawText);
    $tmpfname = tempnam("/tmp", "blank");
    $handle = fopen($tmpfname, "w");
    fwrite($handle, $decodedCode);
    fclose($handle);
    ob_start(function($buffer) use ($tmpfname, $decodedCode) {
        return str_replace($tmpfname, "<pre>" . htmlentities($decodedCode). "</pre>", $buffer);
    });
    include $tmpfname;
    ob_end_flush();
    unlink($tmpfname);
}
```

Now we can use:

```php
$text = encodeText('<?php echo "Gonzalo";' . "\n" . ' print_r(array(1,2,311)); ?>');
includeFromRaw($text);
```

Or if we save our code with the name "Blank.clean"

```php
includeFromFile("Blank.clean");
```

Now if you are brave enough you can write code in our new blankAndSpaces programming language, instead of PHP :)
