---
layout: post
title: "How to configure Symfony's Service Container to use Twitter API"
date: "2013-02-04"
tags: 
  - "dic"
  - "guzzle"
  - "software"
  - "dependency-injection"
  - "php"
  - "symfony2"
  - "technology"
  - "technology"
  - "twitter"
---

Keeping on with the series about Symfony's Services container (another posts [here](http://gonzalo123.com/2012/09/03/dependency-injection-containers-with-php-when-pimple-is-not-enough/) and [here](http://gonzalo123.com/2013/01/14/handling-several-dbal-database-connections-in-symfony2-through-the-dependency-injection-container-with-php/)), now we will use the service container to use Twitter API from a service.

To use Twitter API we need to handle http requests. I've written several post about http request with PHP ([example1](http://gonzalo123.com/2011/08/01/building-a-client-for-a-rest-api-with-php/ "Building a client for a REST API with PHP."), [example2](http://gonzalo123.com/2010/02/06/building-a-rest-client-with-asynchronous-calls-using-php-and-curl/ "Building a REST client with asynchronous calls using PHP and curl")), but today we will use one amazing library to build clients: [Guzzle](http://guzzlephp.org/ "Guzzle"). Guzzle is amazing. We can easily build a Twitter client with it. There's one example is its landing page:

```php
<?php
$client = new Client('https://api.twitter.com/{version}', array('version' => '1.1'));
$oauth  = new OauthPlugin(array(
    'consumer_key'    => '***',
    'consumer_secret' => '***',
    'token'           => '***',
    'token_secret'    => '***'
));
$client->addSubscriber($oauth);
 
echo $client->get('/statuses/user_timeline.json')->send()->getBody();
```

If we are working within a Symfony2 application or a PHP application that uses the Symfony's Dependency injection container component you can easily integrate this simple script in the service container. I will show you the way that I use to do it. Let's start:

The idea is simple. First we include guzzle within our composer.json and execute composer update:

```json
"require": {
    "guzzle/guzzle":"dev-master"
}
```

Then we will create two files, one to store our Twitter credentials and another one to configure the service container:

```yaml
# twitter.conf.yml
parameters:
  twitter.baseurl: https://api.twitter.com/1.1
 
  twitter.config:
    consumer_key: ***
    consumer_secret: ***
    token: ***
    token_secret: ***
```

```yaml
# twitter.yml
parameters:
  class.guzzle.response: Guzzle\Http\Message\Response
  class.guzzle.client: Guzzle\Http\Client
  class.guzzle.oauthplugin: Guzzle\Plugin\Oauth\OauthPlugin
 
services:
  guzzle.twitter.client:
    class: %class.guzzle.client%
    arguments: [%twitter.baseurl%]
    calls:
      - [addSubscriber, [@guzzle.twitter.oauthplugin]]
 
  guzzle.twitter.oauthplugin:
    class: %class.guzzle.oauthplu
```

And finally we include those files in our services.yml:

```yaml
# services.yml
imports:
- { resource: twitter.conf.yml }
- { resource: twitter.yml }
```

And that's all. Now we can use the service without problems:

```php
<?php
 
namespace Gonzalo123\AppBundle\Controller;
 
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
 
class DefaultController extends Controller
{
    public function indexAction($name)
    {
        $twitterClient = $this->container->get('guzzle.twitter.client');
        $status = $twitterClient->get('statuses/user_timeline.json')
             ->send()->getBody();
 
        return $this->render('AppBundle:Default:index.html.twig', array(
            'status' => $status
        ));
    }
}
```
