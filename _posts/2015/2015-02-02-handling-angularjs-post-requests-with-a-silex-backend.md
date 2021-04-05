---
layout: post
title: "Handling AngularJs POST requests with a Silex Backend"
date: "2015-02-02"
tags:
  - "angularjs"
  - "technology"
  - "angular"
  - "php"
  - "silex"
  - "symfony"
---

This days I working a lot with AngularJs applications (who doesn't?). Normally my backend is a Silex application. It's pretty straightforward to build a REST api with Silex. But when we play with an AngularJs client we need to face with a a problem. POST requests "doesn't" work. That's not 100% true. They work, indeed, but they speak different languages.

Silex assumes our POST requests are encoded as application/x-www-form-urlencoded, but angular encodes POST requests as application/json. That's not a problem. It isn't mandatory to use one encoder or another.

For example `name: Gonzalo surname: Ayuso`

If we use application/x-www-form-urlencoded, it's encoded to: `name=Gonzalo&surname=Ayuso`

And if we use application/json, it's encoded to: `{ "name" : "Gonzalo", "surname" : "Ayuso" }`

It's the same but it isn't.

Imagine this Silex example. 

```php
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
 
$app = new Application();
 
$app->post("/post", function (Application $app, Request $request) {
    return $app->json([
        'status' => true,
        'name'   => $request->get('name')
    ]);
});
```

This example works with application/x-www-form-urlencoded but it doesn't work with application/json. We cannot use Symfony\\Component\HttpFoundation\Request parameter's bag as usual. We can get the raw request body with:

```php
$request->getContent();
```

Our content in a application/json encoded Request is a JSON, so we can use json\_decode to access to those parameters.

If we read the Silex documentation we can see how to handle those requests

http://silex.sensiolabs.org/doc/cookbook/json\_request\_body.html

In this post we're going to enclose this code within a service provider. OK, that's not really a service provider (it doesn't provide any service). It just change the request (when we get application/json) without copy and paste the same code within each project.

```php
use Silex\Application;
use G\AngularPostRequestServiceProvider;
use Symfony\Component\HttpFoundation\Request;
 
$app = new Application();
$app->register(new AngularPostRequestServiceProvider());
 
$app->post("/post", function (Application $app, Request $request) {
    return $app->json([
        'status' => true,
        'name'   => $request->get('name')
    ]);
});
```

The service provider is very simple 

```php
namespace G;
 
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
use Pimple\ServiceProviderInterface;
use Pimple\Container;
 
class AngularPostRequestServiceProvider implements ServiceProviderInterface
{
    public function register(Container $app)
    {
        $app->before(function (Request $request) {
            if ($this->isRequestTransformable($request)) {
                $transformedRequest = $this->transformContent($request->getContent());
                $request->request->replace($transformedRequest);
            }
        });
    }
 
    public function boot(Application $app)
    {
    }
 
    private function transformContent($content)
    {
        return json_decode($content, true);
    }
 
    private function isRequestTransformable(Request $request)
    {
        return 0 === strpos($request->headers->get('Content-Type'), 'application/json');
    }
}
```

You can see the whole code in my [github](https://github.com/gonzalo123/AngularPostRequestServiceProvider) account and also in [packagist](https://packagist.org/packages/gonzalo123/angularpostrequestserviceprovider)
