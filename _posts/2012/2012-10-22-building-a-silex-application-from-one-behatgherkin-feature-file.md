---
layout: post
title: "Building a Silex application from one Behat/Gherkin feature file"
date: "2012-10-22"
tags: 
  - "php"
  - "technology"
  - "behat"
  - "gherkin"
  - "silex"
  - "technology"
---

Last days I've playing with [Behat](http://behat.org/). Behat is a behavior driven development (BDD) framework based on Ruby's [Cucumber](http://cukes.info/). Basically with Behat we defenie features within one feature file. I'm not going to crate a Behat tutorial (you can read more about Behat [here](http://docs.behat.org/quick_intro.html)). Behat use Gherkin to write the features files. When I was playing with Behat I had one idea. The idea is simple: **Can we use Gherking to build a Silex application?**. It was a good excuse to study Gherking, indeed ;).

Here comes the feature file:

```yaml
Feature: application API
  Scenario: List users
    Given url "/api/users/list.json"
    And request method is "GET"
    Then instance "\Api\Users"
    And execute function "listUsers"
    And format output into json
 
  Scenario: Get user info
    Given url "/api/user/{userName}.json"
    And request method is "GET"
    Then instance "\Api\User"
    And execute function "info"
    And format output into json
 
  Scenario: Update user information
    Given url "/api/user/{userName}.json"
    And request method is "POST"
    Then instance "\Api\User"
    And execute function "update"
    And format output into json
```

Our API use this simple library:

```php
<?php
 
namespace Api;
use Symfony\Component\HttpFoundation\Request;
 
class User
{
    private $request;
 
    public function __construct(Request $request)
    {
        $this->request = $request;
    }
 
    public function info()
    {
        switch ($this->request->get('userName')) {
            case 'gonzalo':
                return array('name' => 'Gonzalo', 'surname' => 'Ayuso');
            case 'peter':
                return array('name' => 'Peter', 'surname' => 'Parker');
        }
    }
 
    public function update()
    {
        return array('infoUpdated');
    }
}
```

```php
<?php
 
namespace Api;
use Symfony\Component\HttpFoundation\Request;
 
class Users
{
    public function listUsers()
    {
        return array('gonzalo', 'peter');
    }
}
```

The idea is simple. Parse the feature file with behat/gherkin component and create a silex application. And here comes the "magic". This is a simple working prototype, just an experiment for a rainy sunday.

```php
<?php
include __DIR__ . '/../vendor/autoload.php';
define(FEATURE_PATH, __DIR__ . '/api.feature');
 
use Behat\Gherkin\Lexer,
        Behat\Gherkin\Parser,
        Behat\Gherkin\Keywords\ArrayKeywords,
        Behat\Gherkin\Node\FeatureNode,
        Behat\Gherkin\Node\ScenarioNode,
        Symfony\Component\HttpFoundation\Request,
        Silex\Application;
 
$keywords = new ArrayKeywords([
    'en' => [
        'feature' => 'Feature',
        'background' => 'Background',
        'scenario' => 'Scenario',
        'scenario_outline' => 'Scenario Outline',
        'examples' => 'Examples',
        'given' => 'Given',
        'when' => 'When',
        'then' => 'Then',
        'and' => 'And',
        'but' => 'But'
    ],
]);
 
function getMatch($subject, $pattern) {
    preg_match($pattern, $subject, $matches);
    return isset($matches[1]) ? $matches[1] : NULL;
}
 
$app = new Application();
 
function getScenarioConf($scenario) {
    $silexConfItem = [];
 
    /** @var $scenario  ScenarioNode */
    foreach ($scenario->getSteps() as $step) {
        $route = getMatch($step->getText(), '/^url "([^"]*)"$/');
 
        if (!is_null($route)) {
            $silexConfItem['route'] = $route;
        }
 
        $requestMethod = getMatch($step->getText(), '/^request method is "([^"]*)"$/');
        if (!is_null($requestMethod)) {
            $silexConfItem['requestMethod'] = strtoupper($requestMethod);
        }
 
        $instance = getMatch($step->getText(), '/^instance "([^"]*)"$/');
        if (!is_null($instance)) {
            $silexConfItem['className'] = $instance;
        }
 
        $method = getMatch($step->getText(), '/^execute function "([^"]*)"$/');
        if (!is_null($method)) {
            $silexConfItem['method'] = $method;
        }
 
        if ($step->getText() == 'format output into json') {
            $silexConfItem['jsonEncode'] = TRUE;
        }
    }
    return $silexConfItem;
}
 
/** @var $features FeatureNode */
$features = (new Parser(new Lexer($keywords)))->parse(file_get_contents(FEATURE_PATH), FEATURE_PATH);
 
foreach ($features->getScenarios() as $scenario) {
    $silexConfItem = getScenarioConf($scenario);
    $app->match($silexConfItem['route'], function (Request $request) use ($app, $silexConfItem) {
            function getConstructorParams($rClass, $request) {
                $parameters =[];
                foreach ($rClass->getMethod('__construct')->getParameters() as $parameter) {
                    if ('Symfony\Component\HttpFoundation\Request' == $parameter->getClass()->name) {
                        $parameters[$parameter->getName()] = $request;
                    }
                }
                return $parameters;
            }
 
            $rClass = new ReflectionClass($silexConfItem['className']);
            $obj = ($rClass->hasMethod('__construct')) ?
                    $rClass->newInstanceArgs(getConstructorParams($rClass, $request)) :
                    new $silexConfItem['className'];
 
            $output = $obj->{$silexConfItem['method']}();
 
            return ($silexConfItem['jsonEncode'] === TRUE) ? $app->json($output, 200) : $output;
        }
    )->method($silexConfItem['requestMethod']);
}
 
$app->run();
```

You can see the source code in [github](https://github.com/gonzalo123/gherking-sylex). What do you think?
