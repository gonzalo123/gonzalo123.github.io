---
layout: post
title: "Building a client for a REST API with PHP."
date: "2011-08-01"
tags: 
  - "php"
  - "technology"
  - "api"
---

Today we're going to create a library to use a simple RESTfull API for a great project called [Gzaas](http://gzaas.com/). Don't you know what the hell is Gzaas? Gzaas is simple idea idea of a friend of mine (hi [@ojoven](http://twitter.com/ojoven)!). A simple project that creates cool mesages on full screen. Then you can share the link in twitter, facebook and things like that. Yes. I now when I explain it, the first word that appear in our minds is WTF, but it's cool, really [cool](http://gzaas.com/GG0wcn).

Ok. The API is a simple RESTfull [API](http://gzaas.com/project/api-embed/api-general-overview/), so we can use it with a simple curl interface. A few lines of PHP and it will run smoothly. But Gzaas is cool so we're going to create a cool interface too. This days I'm involved into TDD world, so we're going to create the API wrapper using TDD. Let's start.

The first version of Gzaas API is a very simple one. It allow to create new messages, and we also can list the whole set of styles, patterns and fonts availables. The idea is build something as modular as we can. Problably (I hope so) Gzaas will include new features in a future, so we don't want to build a monolitc library. We are going to start with fonts. As we can see in [documentation](http://gzaas.com/project/api-embed/fonts/) we need to perform a GET request to get the full list of available fonts. There's an important parameter (nowFeatured) to show only the featured fonts (the cool ones).

```php
use Gzaas\Api;
$fonts = Api\Fonts::factory()->getAll(Api::FEATURED);
```

So we are going to crate a simple test to test our font API.

```php
$this->assertTrue(count($fonts) > 0);
```

With this test we're going to test fonts, but we also can test styles and patterns.

```php
public function testFonts()
{
    $fonts = Api\Fonts::factory()->getAll(Api::FEATURED);
    $this->assertTrue(count($fonts) > 0);
}
 
public function testStyles()
{
    $styles = Api\Styles::factory()->getAll(Api::FEATURED);
    $this->assertTrue(count($styles) > 0);
}
 
public function testPatterns()
{
    $patterns = Api\Patterns::factory()->getAll(Api::FEATURED);
    $this->assertTrue(count($patterns) > 0);
}
```

Ok now we are going to test the creation of a gzaas. We will use a random font, style and pattern:

```php
$font    = array_rand((array) Api\Fonts::factory()->getAll(Api::FEATURED));
$style   = array_rand((array) Api\Styles::factory()->getAll(Api::FEATURED));
$pattern = array_rand((array) Api\Patterns::factory()->getAll(Api::FEATURED));
```

Let's go:

```php
$gzaas = new Api();
$url = $gzaas->setApiKey('mySuperSecretApiKey') // Get one valid at http://gzaas.com/project/api-embed/api-key
    ->setFont($font)
    ->setBackPattern($pattern)
    ->setStyle($style)
    ->setColor('444444')
    ->setBackcolor('fcfcee')
    ->setShadows('1px 0 2px #ccc')
    ->setVisibility(0)
    ->setLauncher("testin gzaas API");
```

As we can see our library has a fluid interface (I like fluid interface). It's easy to implement. We only need to return the instance of the object (return $this;) in each setter function.

Gzaas API returns the url of the created gzaas when success, so $url variable must be different than empty string. Let's test it:

```php
$this->assertTrue($url != '');
```
To use this library we need to include the following files:

```php
require_once __DIR__ . '/Api/ApiInterface.php'; 
require_once __DIR__ . '/Api/Network.php'; 
require_once __DIR__ . '/Api/ApiAbstract.php'; 
require_once __DIR__ . '/Api/Fonts.php'; 
require_once __DIR__ . '/Api/Patterns.php'; 
require_once __DIR__ . '/Api/Styles.php'; 
require_once __DIR__ . '/Api.php';
```

because of that there's a simple bootstrap.php file to include all files. But we're going to take a step forward. We're going to create [phar](http://php.net/manual/en/book.phar.php) archive with the whole library inside. The creation of phar archive is very simple. We need a index.php file with the bootstrap (it must end with a __HALT_COMPILER();)

```php
include dirname(__FILE__) . '/Api/ApiInterface.php';
include dirname(__FILE__) . '/Api/Network.php';
include dirname(__FILE__) . '/Api/ApiAbstract.php';
include dirname(__FILE__) . '/Api.php';
include dirname(__FILE__) . '/Api/Fonts.php';
include dirname(__FILE__) . '/Api/Patterns.php';
include dirname(__FILE__) . '/Api/Styles.php';
 
__HALT_COMPILER();
```

and the program buildPhar will create our phar file:

```commandline
#!/usr/bin/env php
buildFromDirectory($location);
```

And that's all. Now we can use the bootstrap.php file to include the library or we can also include the phar file:

```php
include 'Gzaas/Gzaas.phar';
```

You can fork the code at github [here](https://github.com/gonzalo123/GzaasApi). Feel free to browse the code and tell us what do you think?
