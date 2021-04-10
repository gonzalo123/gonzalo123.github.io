---
layout: post
title: "PHP Template Engine Comparison. Part 2 (versus Plain PHP)"
date: "2011-01-24"
tags: 
  - "haanga"
  - "php"
  - "technology"
---

In my [last](http://gonzalo123.wordpress.com/2011/01/17/php-template-engine-comparison/) post I created a small (and very personal) test of different templating engines in PHP (Smarty, Haanga and Twiw). Someone posted a comment asking for the comparison between those template engines and old school phtml. OK I’ve done some tests. It’s a bit difficult to create the template inheritance (without cloning one of the template engines and creating a new one) so I have one approximation with a simple include. It’s not the same but it’s similar. I’ve also implemented cycle filter (OK I’ve borrowed getCycle function from Twig. Not really implemented ;) )

Using plain PHP files as template is a big temptation. They're easy to implement and fast. But you need to take care of security issues. For example if you work together with designers, using a template engine is a good solution. It sandboxes our templates and separates the business logic from the HTML. With plain PHP files is very easy to put business logic within templates.

Here we are my examples:

'''php
<?php  /* PHP plain. indexFull.phtml */?>
<html>
    <head>
        <title><?= $title ?></title>
    </head>
    <body>
        <h2>An example with <?= ucwords(strtolower($title)) ?></h2>
        <b>Table with <?= $number ?> rows</b>
        <table>
            <?php foreach ($table as $row) { ?>
            <tr bgcolor="<?= cycle(array('#aaaaaa', '#ffffff'), $row['id']) ?>">
                <td><?= $row['id'] ?></td>
                <td><?= $row['name'] ?></td>
            </tr>
            <?php } ?>
        </table>
    </body>
</html>
'''

and the PHP:

```php
$time = microtime(TRUE);
$mem = memory_get_usage();
 
define('BASE_DIR', dirname(__file__));
 
class DirtyTemplateEngine
{
    protected $_templateDir = null;
    public function __construct($path)
    {
        $this->_templateDir = $path;
    }
 
    function display($template,array $vars) {
        extract($vars);
        ob_start();
 
        function cycle($values, $i)
        {
            if (!is_array($values) && !$values instanceof ArrayAccess) {
                return $values;
            }
 
            return $values[$i % count($values)];
        }
        include($this->_templateDir . '/' . $template);
        echo ob_get_clean();
    }
}
 
$tpl = new DirtyTemplateEngine(BASE_DIR . "/php/templates");
 
$rows = 50;
$data = array();
for ($i = 0; $i < $rows; $i++) {
    $data[] = array('id' => $i, 'name' => "name {$i}");
}
 
$tpl->display('indexFull.phtml', array(
    'number' => $rows,
    'title'  => 'php',
    'table'  => $data
));
 
print_r(array('memory' => (memory_get_usage() - $mem) / (1024 * 1024), 'seconds' => microtime(TRUE) - $time));
```

And with template “inheritance” I know that maybe this test is not comparable with “real” template inheritance

```php
<?php  /* PHP plain. index.phtml */?>
<html>
    <head>
        <title><?= $title ?></title>
    </head>
    <body>
        <h2>An example with <?= ucwords(strtolower($title)) ?></h2>
        <b>Table with <?= $number ?> rows</b>
        <?php include 'block.phtml' ?>;
    </body>
</html>
```

```php
?php  /* PHP plain. block.phtml */?>
<table>
    <?php foreach ($table as $row) { ?>
    <tr bgcolor="<?= cycle(array('#aaaaaa', '#ffffff'), $row['id']) ?>">
        <td><?= $row['id'] ?></td>
        <td><?= $row['name'] ?></td>
    </tr>
    <?php } ?>
</table>
```

<table border="1" width="100%"><tbody><tr><td>(50 rows<strong>)</strong></td><td style="text-align:left;"><strong>Smarty</strong></td><td><strong>Twig</strong></td><td><strong>Haanga</strong></td><td><strong>PHP</strong></td></tr><tr><td><strong>Simple template</strong></td><td>Memory: 0.684497 Time: 0.023710</td><td>Memory: 0.598434 Time: 0.025444</td><td>Memory: 0.124019 Time: 0.004004</td><td>Mem: 0.026512 Time: 0.001536</td></tr><tr><td><strong>Template Inheritance</strong></td><td>Memory: 0.685134 Time: 0.023761</td><td>Memory: 0.619461 Time: 0.028100</td><td>Memory: 0.133472 Time: 0.005005</td><td>Mem: 0.026512 Time: 0.001536</td></tr></tbody></table>

<table border="1" cellpadding="0" width="100%"><tbody><tr><td><strong>(1000 rows)</strong></td><td><strong>Smarty</strong></td><td><strong>Twig</strong></td><td><strong>Haanga</strong></td><td><strong>PHP</strong></td></tr><tr><td><strong>Simple template</strong></td><td>Memory: 1.222743 Time: 0.094762</td><td>Memory: 1.033226 Time: 0.196187</td><td>Memory: 0.558811 Time: 0.043151</td><td>Mem: 0.576526 Time: 0.028120</td></tr><tr><td><strong>Template Inheritance</strong></td><td>Memory: 1.194095 Time: 0.090528</td><td>Memory: 1.054237 Time: 0.191694</td><td>Memory: 0.646381 Time: 0.044402</td><td>Mem: 0.537937 Time: 0.043996</td></tr></tbody></table>

Haanga really rocks again. The performance is very similar than plain PHP templating. Plain PHP is obviously the fastest but Haanga is almost the same. It’s a pity the lack of complete documentation (it’s very hard look into the source code to find features and examples). As Cesar Rodas (Haanga’s developer) told us in a comment Haanga generates self-contained PHP code. Because of that the performance is close to plain PHP execution.

Another useful links about it [here](http://fabien.potencier.org/article/34/templating-engines-in-php) and [here](http://eliw.wordpress.com/2009/10/07/in-response-to-fabien-potencier-twig-php-templating/).
