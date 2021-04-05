---
layout: post
title: "Token based authentication with Silex Applications"
date: "2014-05-05"
tags: 
  - "authentication"
  - "silex"
  - "token"
  - "php"
  - "symfony"
  - "technology"
---

Imagine this simple Silex application:

```php
use Silex\Application;
 
$app = new Application();
 
$app->get('/api/info', function (Application $app) {
    return $app->json([
        'status' => true,
        'info'   => [
            'name'    => 'Gonzalo',
            'surname' => 'Ayuso'
        ]]);
});
 
$app->run();
```

What happens if we want to use a security layer? We can use [sessions](http://silex.sensiolabs.org/doc/providers/session.html). Sessions are the "standard" way to perform authentication in web applications, but when our application is a PhoneGap/Cordova application that uses a Silex server as API server, sessions aren't the best way. The best way now is a token based authentication. The idea is simple. First we need a valid token. Our API server will give us a valid token if we send valid credentials in a login form. Then we need to send the token with each request (the same way than we send the session cookie with each request).

With Silex we can check this token and validate.

```php
use Silex\Application;
 
$app = new Application();
 
$app->get('/api/info', function (Application $app) {
    $token = $app->get('_token');
     
    // here we need to validate the token ...
 
    return $app->json([
        'status' => true,
        'info'   => [
            'name'    => 'Gonzalo',
            'surname' => 'Ayuso'
        ]]);
});
 
$app->run();
```

It isn't an elegant solution. We need to validate the token within all routes and that's bored. We also can use [middlewares](http://silex.sensiolabs.org/doc/middlewares.html) and validates the token with $app->before(). We're going to build something like this, but with a few variations. First I want to keep the main application as clean as possible. Validation logic must be separated from application logic, so we will extend Silex\\Application. Our main application will be like this:

```php
use G\Silex\Application;
 
$app = new Application();
 
$app->get('/api/info', function (Application $app) {
    return $app->json([
        'status' => true,
        'info'   => [
            'name'    => 'Gonzalo',
            'surname' => 'Ayuso'
        ]]);
});
 
$app->run();
```

Instead of Silex\Application we'll use G\Silex\Application.

```php
namespace G\Silex;
 
use Silex\Application as SilexApplication;
use G\Silex\Provider\Login\LoginBuilder;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
 
class Application extends SilexApplication
{
    public function __construct(array $values = [])
    {
        parent::__construct($values);
 
        LoginBuilder::mountProviderIntoApplication('/auth', $this);
 
        $this->after(function (Request $request, Response $response) {
            $response->headers->set('Access-Control-Allow-Origin', '*');
        });
    }
}
```

Our new G\Silex\Application is a Silex\\Application [enabling CORS](http://gonzalo123.com/2013/12/16/enabling-cors-in-a-restfull-silex-server-working-with-a-phonegapcordova-application/http://). We also mount a Service provider.

The responsibility of our API server will be check the token of every request and to provide one way to get a new token. To get a new token we will create a route "/auth/validateCredentials". If a valid credentials are given, new token will be send to client.

Our Service provider has two parts: a [service provider](http://silex.sensiolabs.org/doc/providers.html#service-providers) and a [controller provider](http://silex.sensiolabs.org/doc/providers.html#controller-providers).

To mount both providers we will use a LoginBuilder class: 

```php
namespace G\Silex\Provider\Login;
 
use Silex\Application;
 
class LoginBuilder
{
    public static function mountProviderIntoApplication($route, Application $app)
    {
        $app->register(new LoginServiceProvider());
        $app->mount($route, (new LoginControllerProvider())->setBaseRoute($route));
    }
}
```

Our Controller provider:

```php
namespace G\Silex\Provider\Login;
 
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;
use Symfony\Component\HttpFoundation\Request;
use Silex\ControllerProviderInterface;
use Silex\Application;
 
class LoginControllerProvider implements ControllerProviderInterface
{
    const VALIDATE_CREDENTIALS = '/validateCredentials';
    const TOKEN_HEADER_KEY     = 'X-Token';
    const TOKEN_REQUEST_KEY    = '_token';
 
    private $baseRoute;
 
    public function setBaseRoute($baseRoute)
    {
        $this->baseRoute = $baseRoute;
 
        return $this;
    }
 
    public function connect(Application $app)
    {
        $this->setUpMiddlewares($app);
 
        return $this->extractControllers($app);
    }
 
    private function extractControllers(Application $app)
    {
        $controllers = $app['controllers_factory'];
 
        $controllers->get(self::VALIDATE_CREDENTIALS, function (Request $request) use ($app) {
            $user   = $request->get('user');
            $pass   = $request->get('pass');
            $status = $app[LoginServiceProvider::AUTH_VALIDATE_CREDENTIALS]($user, $pass);
 
            return $app->json([
                'status' => $status,
                'info'   => $status ? ['token' => $app[LoginServiceProvider::AUTH_NEW_TOKEN]($user)] : []
            ]);
        });
 
        return $controllers;
    }
 
    private function setUpMiddlewares(Application $app)
    {
        $app->before(function (Request $request) use ($app) {
            if (!$this->isAuthRequiredForPath($request->getPathInfo())) {
                if (!$this->isValidTokenForApplication($app, $this->getTokenFromRequest($request))) {
                    throw new AccessDeniedHttpException('Access Denied');
                }
            }
        });
    }
 
    private function getTokenFromRequest(Request $request)
    {
        return $request->headers->get(self::TOKEN_HEADER_KEY, $request->get(self::TOKEN_REQUEST_KEY));
    }
 
    private function isAuthRequiredForPath($path)
    {
        return in_array($path, [$this->baseRoute . self::VALIDATE_CREDENTIALS]);
    }
 
    private function isValidTokenForApplication(Application $app, $token)
    {
        return $app[LoginServiceProvider::AUTH_VALIDATE_TOKEN]($token);
    }
}
```

And our Service provider: 

```php
namespace G\Silex\Provider\Login;
 
use Silex\Application;
use Silex\ServiceProviderInterface;
 
class LoginServiceProvider implements ServiceProviderInterface
{
    const AUTH_VALIDATE_CREDENTIALS = 'auth.validate.credentials';
    const AUTH_VALIDATE_TOKEN       = 'auth.validate.token';
    const AUTH_NEW_TOKEN            = 'auth.new.token';
 
    public function register(Application $app)
    {
        $app[self::AUTH_VALIDATE_CREDENTIALS] = $app->protect(function ($user, $pass) {
            return $this->validateCredentials($user, $pass);
        });
 
        $app[self::AUTH_VALIDATE_TOKEN] = $app->protect(function ($token) {
            return $this->validateToken($token);
        });
 
        $app[self::AUTH_NEW_TOKEN] = $app->protect(function ($user) {
            return $this->getNewTokenForUser($user);
        });
    }
 
    public function boot(Application $app)
    {
    }
 
    private function validateCredentials($user, $pass)
    {
        return $user == $pass;
    }
 
    private function validateToken($token)
    {
        return $token == 'a';
    }
 
    private function getNewTokenForUser($user)
    {
        return 'a';
    }
}
```

Our Service provider will have the logic to validate credentials, token and it must be able to generate a new token:

```php
private function validateCredentials($user, $pass)
{
    return $user == $pass;
}
 
private function validateToken($token)
{
    return $token == 'a';
}
 
private function getNewTokenForUser($user)
{
    return 'a';
}
```

As we can see the logic of the example is very simple. It's just an example and here we must to perform our logic. Probably we need to check credentials with our database, and our token must be stored somewhere to be validated later.

You can see the example in my [github](https://github.com/gonzalo123/token) account. In another post we will see how to build a client application with angularJs to use this API server.
