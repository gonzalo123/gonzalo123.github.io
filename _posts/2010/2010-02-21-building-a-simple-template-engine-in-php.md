---
layout: post
layout: post
title: "Building a simple template engine in PHP."
date: "2010-02-21"
tags: 
  - "php"
categories: 
  - "php"
---

Yes that's another template engine in PHP. I've been using [Smarty](http://www.smarty.net/) for years. It's easy to install and easy to use. But there is something I don't like in Smarty. I need to learn another language (smarty markup) to create my templates. Normally my templates are not very complex. I only use them to move HTML code outside PHP. Basically my templates are: HTML and some variables that pass from PHP to tpl. Sometimes I need to do a loop. I know Smarty has a lot of helpers but I never use them.

Now I'm in a project and I must to choose a template engine. This project don't have any dependencies to any other libraries, so I don't want to include full Smarty library to my simple templates. A quick search in [Google](http://www.google.es/search?sourceid=chrome&ie=UTF-8&q=PHP+template+engine) gives us a list of template engines in PHP. Mostly all engines use PHP as template language. That's a good decision. In fact PHP is a template language so, why we need to use another one?. I think the main problem of using PHP as template language is the temptation of put logic in the template and have a nice spaghetti code.

So I am going to build a simple template engine. Let's start: As always I like to start from the interface. When I start a library I like to think the library is finished (before start coding. cool isn't it?) and write the code to use it. Finally when the I like the interface I start coding the library.

I want this template:

```php
<h1>Example of tpl</h1>
var1 = _('var1') ?>
var2 = _('var2') ?>
foreach ($this->_('var3') as $item) {
    echo $this->clean($item);
}
?>
```

And I want to call it with something like this:

```php
echo Tpl::singleton()->init('demo1.phtml')->render(array(
    'var1' => 1,
    'var2' => 2,
    'var3' => array(1, 2, 3, 4, 5, 6, 7)
    ));
```
Basically Tpl class will be a container to collect the configuration and init function will be a factory of another class (Tpl\_Instance) that will do the templating itself.

render function is the main function. Basically call to php's include function with the selected tpl.

```php
if (!is_file($_tplFile)) {
    throw new Exception('Template file not found');
}
 
ini_set('implicit_flush',false);
ob_start();
include ($_tplFile);
 
$out = ob_get_contents();
ob_end_clean();
ini_set('implicit_flush',true);
 
return $out;
```

As we see the tpl file is a simple PHP file included in our script with Tpl\_Instance::render. So $this in our tpl's PHP code is the instance of Tpl\_Instance. That means whe can use protected and even private functions of Tpl\_Instance.

Now I going to show different usages of the library:

```php
// Sets the path of templates. If nuls asumes file is absolute
Tpl::singleton()->setConf(Tpl::TPL_DIR, realpath(dirname(__FILE__)));
echo Tpl::singleton()->init('demo1.phtml')->render(array(
    'var1' => 1,
    'var2' => 2,
    'var3' => array(1, 2, 3, 4, 5, 6, 7)
    ));
```

```php
// The same instance a different template and params added in a different way
$tpl = Tpl::singleton()->init('demo2.phtml');
$tpl->addParam('header', 'header');
$tpl->addParam('footer', 'footer');
echo $tpl->render();
```

```php
// Disable exceptions if we don't assign a variable
Tpl::singleton()->setConf(Tpl::THROW_EXCEPTION_WITH_PARAMS, false);
$tpl = Tpl::singleton()->init('demo1.phtml');
$tpl->addParam('var1', 'aaaa');
$tpl->addParam('var3', array(1, 2, 3, 4, 5, 6, 7));
echo $tpl->render();
```

```php
// Using factory
$objTpl = Tpl::factory();
$objTpl->setConf(Tpl::THROW_EXCEPTION_WITH_PARAMS, true);
try {
    $tpl = $objTpl->init('demo1.phtml');
    $tpl->addParam('var1', 'aaaa');
    $tpl->addParam('var3', array(1, 2, 3, 4, 5, 6, 7));
    echo $tpl->render();
} catch (Exception $e) {
    echo "" . $e->getMessage() . "
 
";
}
```

And like always full source code is available on [google code](http://code.google.com/p/gam-tpl/).
