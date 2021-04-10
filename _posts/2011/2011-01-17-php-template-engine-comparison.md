---
layout: post
title: "PHP Template Engine Comparison"
date: "2011-01-17"
tags: 
  - "haanga"
  - "php"
  - "technology"
  - "twig"
---

I’m going to face a project using a template engine with PHP. Because of that I’m will perform a small benchmark test of several PHP template engines. That’s not an exhaustive performance test. It’s only my personal test. Template engines has a lot of features but I normally only use a few of them and the other features very seldom. In this performance test I will check the same features under different template engines to see the syntax differences and the performance. The template engines selected for the test are [Smarty](http://www.smarty.net/), [Twig](http://www.twig-project.org/) and [Haanga](http://haanga.org/). Let’s start:

**Smarty. v3.0.6** 
It’s probably the most famous template engine. It’s a mature project. For years it was “the” template engine and the others were the “alternatives”. It was famous because of the speed.

**Twig. v1.0.0-RC1-8** 
It’s a new template engine developed by [Fabien Potencier](http://fabien.potencier.org/), the creator of the [symfony](http://www.symfony-project.org/) framework. One of the PHP's rock stars nowadays. It’s going to be an important part of the new [symfony 2.0](http://symfony-reloaded.org/) framework. Twig borrows the template syntax from [Django](http://www.djangoproject.com/) (probably the main web framework if we work with Python)

**Haanga. v1.0.4-14** 
It’s another new template engine using the Django style. It was developed for [Menéame](http://www.meneame.net/) by [César Rodas](http://cesarodas.com/).

I’ve decided to create two tests. One with a simple template and another using template Inheritance. The both cases renders one html page using one variable, filters and for loop to create an HTML table. Basically I’ve created those test because they're the things I normally use. I will run the test with an HTML table of 50 rows and 1000 rows

# Simple template

## Smarty

```html
{* Smarty. indexFull.tpl*}
<html>
    <head>
        <title>{$title}</title>
    </head>
    <body>
        <h2>An example with {$title|capitalize}</h2>
        <b>Table with {$number|escape} rows</b>
        <table>
{foreach $table as $row}
            <tr bgcolor="{cycle values="#aaaaaa,#ffffff"}">
                <td>{$row.id}</td>
                <td>{$row.name}</td>
            </tr>
{foreachelse}
            <tr><td>No items were found</td></tr>
{/foreach}
        </table>
    </body>
</html>
```

And the PHP conde:

```php
// index.php
$time = microtime(TRUE);
$mem = memory_get_usage();
 
define('BASE_DIR', dirname(__file__));
require(BASE_DIR . '/include/smarty/Smarty.class.php');
 
$smarty = new Smarty();
 
$smarty->setTemplateDir(BASE_DIR . '/smarty/templates');
$smarty->setCompileDir(BASE_DIR . '/smarty/templates_c');
$smarty->setCacheDir(BASE_DIR . '/smarty/cache');
$smarty->setConfigDir(BASE_DIR .'/smarty/configs');
 
$smarty->assign('title', "smarty");
 
$rows = 1000;
$data = array();
for ($i=0; $i<$rows; $i++ ) {
    $data[] = array('id' => $i, 'name' => "name {$i}");
}
$smarty->assign('table', $data);
$smarty->assign('number', $rows);
$smarty->display('indexFull.tpl');
 
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'seconds' => microtime(TRUE) - $time));
```

## Twig

```html
{#  Twig. indexFull.html #} 
<html>
    <head>
        <title>{{ title }}</title>
    </head>
    <body>
        <h2>An example with {{ title|title }}</h2>
        <b>Table with {{ number|escape}} rows</b>
        <table>
            {% for row in table %}
            <tr bgcolor="{{ cycle(['#aaaaaa', '#ffffff'], row.id) }}">
                <td>{{ row.id }}</td>
                <td>{{ row.name }}</td>
            </tr>
            {% endfor %}
        </table>
    </body>
</html>
```

And the PHP code:

```php
// index.php
$time = microtime(TRUE);
$mem = memory_get_usage();
 
define('BASE_DIR', dirname(__file__));
require_once BASE_DIR . '/include/Twig/Autoloader.php';
Twig_Autoloader::register();
 
$loader = new Twig_Loader_Filesystem(BASE_DIR . '/twig/templates');
$twig = new Twig_Environment($loader, array(
            'cache' => BASE_DIR . '/twig/compiled',
            'auto_reload' => true
        ));
$template = $twig->loadTemplate('indexFull.html');
 
$rows = 1000;
$data = array();
for ($i = 0; $i < $rows; $i++) {
    $data[] = array('id' => $i, 'name' => "name {$i}");
}
 
$template->display(array(
    'number' => $rows,
    'title'  => 'twig',
    'table'  => $data
));
 
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'seconds' => microtime(TRUE) - $time));

```

## Haanga

```html
{#  Haanga. indexFull.html #} 
<html>
    <head>
        <title>{{ title }}</title>
    </head>
    <body>
        <h2>An example with {{ title|title }}</h2>
        <b>Table with {{ number|escape}} rows</b>
        <table>
            {% for row in table %}
            <tr bgcolor="{% cycle '#aaaaaa' '#ffffff' %}">
                <td>{{ row.id }}</td>
                <td>{{ row.name }}</td>
            </tr>
            {% endfor %}
        </table>
    </body>
</html>
```

And the PHP code:

```php
// index.php
$time = microtime(TRUE);
$mem = memory_get_usage();
 
define('BASE_DIR', dirname(__file__));
require(BASE_DIR . '/include/Haanga.php');
 
Haanga::configure(array(
    'template_dir' => BASE_DIR . '/haanga/templates',
    'cache_dir' => BASE_DIR . '/haanga/compiled',
));
 
$rows = 1000;
$data = array();
for ($i=0; $i<$rows; $i++ ) {
    $data[] = array('id' => $i, 'name' => "name {$i}");
}
Haanga::Load('indexFull.html', array(
    'number' => $rows,
    'title'  => 'haanga',
    'table'  => $data
    ));
 
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'seconds' => microtime(TRUE) - $time));

```

# With template Inheritance

With this test I use the same php file, changing template name from indexFull to index.

## Smarty

```html
{* Smarty. index.tpl*}
{extends file="layout.tpl"}
{block name=table}
<table>
{foreach $table as $row}
    <tr bgcolor="{cycle values="#aaaaaa,#ffffff"}">
        <td>{$row.id}</td>
        <td>{$row.name}</td>
    </tr>
{foreachelse}
    <tr><td>No items were found</td></tr>
{/foreach}
</table>
{/block}
```

```html
{* Smarty. layout.tpl*}
<html>
    <head>
        <title>{$title}</title>
    </head>
    <body>
        <h2>An example with {$title|capitalize}</h2>
        <b>Table with {$number|escape} rows</b>
        {block name=table}{/block}
    </body>
</html>
```

## Twig

```html
{#  Twig. index.html #} 
{extends "layout.html" %}
{\% block table %}
<table>
    {% for row in table %}
    <tr bgcolor="{{ cycle(['#aaaaaa', '#ffffff'], row.id) }}">
        <td>{{ row.id }}</td>
        <td>{{ row.name }}</td>
    </tr>
    {% else %}
    <tr><td>No items were found</td></tr>
    {% endfor %}
</table>
{\% endblock %}
```

```html
{#  Twig. layout.html #} 
<html>
    <head>
        <title>{{ title }}</title>
    </head>
    <body>
        <h2>An example with {{ title|title }}</h2>
        <b>Table with {{ number|escape}} rows</b>
        {\% block table %}{\% endblock %}
    </body>
</html>
```

## Haanga

```html
{\% extends "layout.html" %}
{#  Haanga. index.html #} 
{\% block table %}
<table>
    {% for row in table %}
    <tr bgcolor="{% cycle '#aaaaaa' '#ffffff' %}">
        <td>{{ row.id }}</td>
        <td>{{ row.name }}</td>
    </tr>
    {% endfor %}
</table>
{\% endblock %}
```

```html
{#  Haanga. layout.html #} 
<html>
    <head>
        <title>{{ title }}</title>
    </head>
    <body>
        <h2>An example with {{ title|title }}</h2>
        <b>Table with {{ number|escape}} rows</b>
        {\% block table %}{\% endblock %}
    </body>
</html>
```

# Outcomes of the tests:

<table border="1" width="100%"><tbody><tr><td>(50 rows<strong>)</strong></td><td style="text-align:left;"><strong>Smarty</strong></td><td><strong>Twig</strong></td><td><strong>Haanga</strong></td></tr><tr><td><strong>Simple template</strong></td><td>Memory: 0.684497 Time: 0.023710</td><td>Memory: 0.598434 Time: 0.025444</td><td>Memory: 0.124019 Time: &nbsp;0.004004</td></tr><tr><td><strong>Template Inheritance</strong></td><td>Memory: 0.685134 Time: 0.023761</td><td>Memory: 0.619461 Time: 0.028100</td><td>Memory: 0.133472 Time: 0.005005</td></tr></tbody></table>

<table border="1" cellpadding="0" width="100%"><tbody><tr><td><strong>(1000 rows)</strong></td><td><strong>Smarty</strong></td><td><strong>Twig</strong></td><td><strong>Haanga</strong></td></tr><tr><td><strong>Simple template</strong></td><td>Memory: 1.222743 Time: 0.094762</td><td>Memory: 1.033226 Time: 0.196187</td><td>Memory: 0.558811 Time: 0.043151</td></tr><tr><td><strong>Template Inheritance</strong></td><td>Memory: 1.194095 Time: 0.090528</td><td>Memory: 1.054237 Time: 0.191694</td><td>Memory: 0.646381 Time: 0.044402</td></tr></tbody></table>

Haanga really rocks in the test. It’s the fastest in all cases and it’s the best using memory. The main problem I’ve seen with Haanga is the lack of documentation. When I wanted to use the cycle filter (to create the zebra style in the HTML table) I didn’t find anything about it. I had to browse the source code and finally I found it in one tests. Whereas Smarty documentation is brilliant and Twig is good enough.

The HTML template syntax is almost the same with Twig and Haanga (in fact both of them are Django style). Smarty is a bit different but is very similar. The PHP part in Smarty looks like a bit old fashioned compared with Haanga and Twig, but it's really easy to use.

The performance of Twig and Smarty are similar. Twig is slightly better. But with simple templates i’ts almost the same.
