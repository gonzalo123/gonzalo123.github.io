---
layout: post
title: "Google App Engine, PHP and Silex. Setting up a Login Provider"
date: "2013-05-20"
tags: 
  - "google-app-engine"
  - "symfony"
  - "technology"
  - "gae"
  - "silex"
---

Last week Google [announced](http://googlecloudplatform.blogspot.com.es/2013/05/ushering-in-next-generation-of.html) the PHP support for Google App Engine (GAE). PHPStorm, the great IDE for PHP development, also [announced](http://blog.jetbrains.com/phpstorm/2013/05/support-for-google-app-engine-php-in-phpstorm/) support for Google App Engine PHP. Because of that now is time to hack a little bit with this new toy.

I've worked in a couple of projects with Google App Engine in the past (with Python). With PHP the process is almost the same. First we need to define our application in the app.yaml file. In our example we are going to redirect all requests to main.php, where our Silex application is defined.

```yaml
application: silexgae
version: 1
runtime: php
api_version: 1
threadsafe: true
 
handlers:
- url: .*
  script: main.php
```

To build a simple Silex application over Google App Engine is pretty straightforward (more info [here](https://developers.google.com/appengine/downloads?hl=en#Google_App_Engine_SDK_for_PHP)). Because of that we're going to go a little further. We are going to use the log-in framework provided by GAE to log-in with our Goggle account within our Silex application. In fact we can use the standard OAuth authentication process but Google provides a simple [way](https://developers.google.com/appengine/docs/php/users/loginurls) to use our gmail account.

Now we're going to build a LoginProvider to make this process simpler. Our base Silex application will be the following one: 

```php
<?php
require_once __DIR__ . '/vendor/autoload.php';
 
use Silex\Application;
use Gae\LoginProvider;
use Gae\Auth;
 
$app = new Application();
$app->register(new LoginProvider(), array(
    'auth.onlogin.callback.url' => '/private',
    'auth.onlogout.callback.url' => '/loggedOut',
));
 
/** @var Auth $auth */
$auth = $app['gae.auth']();
 
$app->get('/', function () use ($app, $auth) {
    return $auth->isLogged() ?
        $app->redirect("/private") :
        "<a href='" . $auth->getLoginUrl() . "'>login</a>";
});
 
$app->get('/private', function () use ($app, $auth) {
    return $auth->isLogged() ?
        "Hello " . $auth->getUser()->getNickname() . 
          " <a href='" . $auth->getLogoutUrl() . "'>logout</a>" :
        $auth->getRedirectToLogin();
});
 
$app->get('/loggedOut', function () use ($app) {
    return "Thank you!";
});
 
$app->run();
```

Our LoginProvider is a simple Class that implements Silex\ServiceProviderInterface 

```php
<?php
namespace Gae;
 
require_once 'google/appengine/api/users/UserService.php';
 
use google\appengine\api\users\UserService;
use Gae\Auth;
use Silex\Application;
use Silex\ServiceProviderInterface;
 
class LoginProvider implements ServiceProviderInterface
{
    public function register(Application $app)
    {
        $app['gae.auth'] = $app->protect(function () use ($app) {
            return new Auth($app, UserService::getCurrentUser());
        });
    }
 
    public function boot(Application $app)
    {
    }
}
```

As you can see our Provider class proviedes us an instance of Gae\Auth class 

```php
<?php
namespace Gae;
 
require_once 'google/appengine/api/users/UserService.php';
 
use google\appengine\api\users\User;
use google\appengine\api\users\UserService;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Silex\Application;
 
class Auth
{
    private $user = null;
    private $loginUrl;
    private $logoutUrl;
    private $logged;
 
    public function __construct(Application $app, User $user=null)
    {
        $this->user = $user;
 
        if (is_null($user)) {
            $this->loginUrl = UserService::createLoginUrl($app['auth.onlogin.callback.url']);
            $this->logged = false;
        } else {
            $this->logged = true;
            $this->logoutUrl = UserService::createLogoutUrl($app['auth.onlogout.callback.url']);
        }
    }
 
    /**
     * @return RedirectResponse
     */
    public function getRedirectToLogin()
    {
        return new RedirectResponse($this->getLoginUrl());
    }
    /**
     * @return boolean
     */
    public function isLogged()
    {
        return $this->logged;
    }
 
    /**
     * @return string
     */
    public function getLoginUrl()
    {
        return $this->loginUrl;
    }
 
    /**
     * @return string
     */
    public function getLogoutUrl()
    {
        return $this->logoutUrl;
    }
 
    /**
     * @return \google\appengine\api\users\User|null
     */
    public function getUser()
    {
        return $this->user;
    }
}
```

And that's all. Full code is available in my [github](https://github.com/gonzalo123/silex-gae-login-provider) account and you can also use [composer](https://packagist.org/packages/gonzalo123/silex-gae-login-provider) to include this provider within your projects.

[![youtube](https://img.youtube.com/vi/hkW-xiSYiak\/0.jpg)](https://www.youtube.com/watch?v=hkW-xiSYiak\)
