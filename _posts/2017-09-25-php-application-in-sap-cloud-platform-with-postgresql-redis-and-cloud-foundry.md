---
layout: post
title: "PHP application in SAP Cloud Platform. With PostgreSQL, Redis and Cloud Foundry"
date: "2017-09-25"
categories: 
  - "laravel"
  - "lumen"
  - "php"
  - "postgresql"
  - "scp"
  - "technology"
tags: 
  - "cloud-foundry"
  - "postgresql"
  - "redis"
  - "sap"
  - "sap-cloud-platform"
  - "silex"
---

Keeping on with my study of [SAP's cloud platform](https://account.hana.ondemand.com/) (SCP) and [Cloud Foundry](https://cloudplatform.sap.com/capabilities/runtimes-containers/cloud-foundry.html) today I'm going to build a simple [PHP](https://docs.cloudfoundry.org/buildpacks/php/index.html) application. This application serves a simple Bootstrap landing page. The application uses a HTTP basic authentication. The credentials are validated against a PostgreSQL database. It also has a API to retrieve the localtimestamp from database server (just for play with a database server). I also want to play with Redis in the cloud too, so the API request will have a Time To Live (ttl) of 5 seconds. I will use a Redis service to do it.

First we create our services in cloud foundry. I'm using the free layer of SAP cloud foundry for this example. I'm not going to explain here how to do that. It's pretty straightforward within SAP's coopkit. Time ago I played with IBM's cloud foundry too. I remember that it was also very simple too.

Then we create our application (.bp-config/options.json) 

```json
{
    "WEBDIR": "www",
    "LIBDIR": "lib",
    "PHP_VERSION": "{PHP_70_LATEST}",
    "PHP_MODULES": ["cli"],
    "WEB_SERVER": "nginx"
}
```

If we want to use our PostgreSQL and Redis services with our PHP Appliacation we need to connect those services to our application. This operation can be done also with SAP's Cockpit.

Now is the turn of PHP application. I normally use Silex framework within my backends, but now there's a problem: [Silex is dead](https://gonzalo123.com/2017/07/10/silex-is-dead-or-not/). I feel a little bit sad but I'm not going to cry. It's just a tool and there're another ones. I've got my example with Silex but, as an exercise, I will also do it with [Lumen](https://lumen.laravel.com/).

Let's start with Silex. If you're familiar with Silex micro framework (or another microframework, indeed) you can see that there isn't anything especial.

```php
use Symfony\Component\HttpKernel\Exception\HttpException;
use Symfony\Component\HttpFoundation\Request;
use Silex\Provider\TwigServiceProvider;
use Silex\Application;
use Predis\Client;
 
if (php_sapi_name() == "cli-server") {
    // when I start the server my local machine vendors are in a different path
    require __DIR__ . '/../vendor/autoload.php';
    // and also I mock VCAP_SERVICES env
    $env   = file_get_contents(__DIR__ . "/../conf/vcap_services.json");
    $debug = true;
} else {
    require 'vendor/autoload.php';
    $env   = $_ENV["VCAP_SERVICES"];
    $debug = false;
}
 
$vcapServices = json_decode($env, true);
 
$app = new Application(['debug' => $debug, 'ttl' => 5]);
 
$app->register(new TwigServiceProvider(), [
    'twig.path' => __DIR__ . '/../views',
]);
 
$app['db'] = function () use ($vcapServices) {
    $dbConf = $vcapServices['postgresql'][0]['credentials'];
    $dsn    = "pgsql:dbname={$dbConf['dbname']};host={$dbConf['hostname']};port={$dbConf['port']}";
    $dbh    = new PDO($dsn, $dbConf['username'], $dbConf['password']);
    $dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $dbh->setAttribute(PDO::ATTR_CASE, PDO::CASE_UPPER);
    $dbh->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
 
    return $dbh;
};
 
$app['redis'] = function () use ($vcapServices) {
    $redisConf = $vcapServices['redis'][0]['credentials'];
 
    return new Client([
        'scheme'   => 'tcp',
        'host'     => $redisConf['hostname'],
        'port'     => $redisConf['port'],
        'password' => $redisConf['password'],
    ]);
};
 
$app->get("/", function (Application $app) {
    return $app['twig']->render('index.html.twig', [
        'user' => $app['user'],
        'ttl'  => $app['ttl'],
    ]);
});
 
$app->get("/timestamp", function (Application $app) {
    if (!$app['redis']->exists('timestamp')) {
        $stmt = $app['db']->prepare('SELECT localtimestamp');
        $stmt->execute();
        $app['redis']->set('timestamp', $stmt->fetch()['TIMESTAMP'], 'EX', $app['ttl']);
    }
 
    return $app->json($app['redis']->get('timestamp'));
});
 
$app->before(function (Request $request) use ($app) {
    $username = $request->server->get('PHP_AUTH_USER', false);
    $password = $request->server->get('PHP_AUTH_PW');
 
    $stmt = $app['db']->prepare('SELECT name, surname FROM public.user WHERE username=:USER AND pass=:PASS');
    $stmt->execute(['USER' => $username, 'PASS' => md5($password)]);
    $row = $stmt->fetch();
    if ($row !== false) {
        $app['user'] = $row;
    } else {
        header("WWW-Authenticate: Basic realm='RIS'");
        throw new HttpException(401, 'Please sign in.');
    }
}, 0);
 
$app->run();
```

Maybe the only especial thing is the way that autoloader is done. We are initializing autoloader in two different ways. One way when the application is run in the cloud and another one when the application is run locally with PHP's built-in server. That's because vendors are located in different paths depending on which environment the application lives in. When Cloud Foundry connect services to appliations it injects environment variables with the service configuration (credentials, host, ...). It uses VCAP\_SERVICES env var.

I use the built-in server to run the application locally. When I'm doing that I don't have VCAP\_SERVICES variable. And also my services information are different than when I'm running the application in the cloud. Maybe it's better with an environment variable but I'm using this trick:

```php
if (php_sapi_name() == "cli-server") {
    // I'm runing the application locally
} else {
    // I'm in the cloud
}
```

So when I'm locally I mock VCAP\_SERVICES with my local values and also, for example, configure Silex application in debug mode.

Sometimes I want to run my application locally but I want to use the cloud services. I cannot connect directly to those services, but we can do it over ssh through our connected application. For example If our PostgreSQL application is running on 10.11.241.0:48825 we can map this remote port (in a private network) to our local port with this command.

```
cf ssh -N -T -L 48825:10.11.241.0:48825 silex
```

You can see more information about this command [here](https://docs.cloudfoundry.org/devguide/deploy-apps/ssh-services.html).

Now we can use pgAdmin, for example, in our local machine to connect to cloud server.

We can do the same with Redis

```
cf ssh -N -T -L 54266:10.11.241.9:54266 silex
```

And basically that's all. Now we'll do the same with Lumen. The idea is create the same application with Lumen instead of Silex. It's a dummy application but it cover task that I normally use. I also will re-use the Redis and PostgreSQL services from the previous project.

```php
use App\Http\Middleware;
use Laravel\Lumen\Application;
use Laravel\Lumen\Routing\Router;
use Predis\Client;
 
if (php_sapi_name() == "cli-server") {
    require __DIR__ . '/../vendor/autoload.php';
    $env = 'dev';
} else {
    require 'vendor/autoload.php';
    $env = 'prod';
}
 
(new Dotenv\Dotenv(__DIR__ . "/../env/{$env}"))->load();
 
$app = new Application();
 
$app->routeMiddleware([
    'auth' => Middleware\AuthMiddleware::class,
]);
 
$app->register(App\Providers\VcapServiceProvider::class);
$app->register(App\Providers\StdoutLogServiceProvider::class);
$app->register(App\Providers\DbServiceProvider::class);
$app->register(App\Providers\RedisServiceProvider::class);
 
$app->router->group(['middleware' => 'auth'], function (Router $router) {
    $router->get("/", function () {
        return view("index", [
            'user' => config("user"),
            'ttl'  => getenv('TTL'),
        ]);
    });
 
    $router->get("/timestamp", function (Client $redis, PDO $conn) {
        if (!$redis->exists('timestamp')) {
            $stmt = $conn->prepare('SELECT localtimestamp');
            $stmt->execute();
            $redis->set('timestamp', $stmt->fetch()['TIMESTAMP'], 'EX', getenv('TTL'));
        }
 
        return response()->json($redis->get('timestamp'));
    });
});
 
$app->run();
```

I've created four servicer providers. One for handle Database connections (I don't like ORMs) 

```php
namespace App\Providers;
 
use Illuminate\Support\ServiceProvider;
use PDO;
 
class DbServiceProvider extends ServiceProvider
{
    public function register()
    {
    }
 
    public function boot()
    {
        $vcapServices = app('vcap_services');
 
        $dbConf = $vcapServices['postgresql'][0]['credentials'];
        $dsn    = "pgsql:dbname={$dbConf['dbname']};host={$dbConf['hostname']};port={$dbConf['port']}";
        $dbh    = new PDO($dsn, $dbConf['username'], $dbConf['password']);
        $dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        $dbh->setAttribute(PDO::ATTR_CASE, PDO::CASE_UPPER);
        $dbh->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
 
        $this->app->bind(PDO::class, function ($app) use ($dbh) {
            return $dbh;
        });
    }
}
```

Another one for Redis. I need to study a little bit more Lumen. I know that Lumen has a built-in tool to work with Redis.

```php
namespace App\Providers;
 
use Illuminate\Support\ServiceProvider;
use Predis\Client;
 
class RedisServiceProvider extends ServiceProvider
{
    public function register()
    {
    }
 
    public function boot()
    {
        $vcapServices = app('vcap_services');
        $redisConf    = $vcapServices['redis'][0]['credentials'];
 
        $redis = new Client([
            'scheme'   => 'tcp',
            'host'     => $redisConf['hostname'],
            'port'     => $redisConf['port'],
            'password' => $redisConf['password'],
        ]);
 
        $this->app->bind(Client::class, function ($app) use ($redis) {
            return $redis;
        });
    }
}
```

And the last one to work with Vcap environment variables. Probably I need to integrate it with dotenv files

```php
namespace App\Providers;
 
use Illuminate\Support\ServiceProvider;
use Monolog;
 
class StdoutLogServiceProvider extends ServiceProvider
{
    public function register()
    {
        app()->configureMonologUsing(function (Monolog\Logger $monolog) {
            return $monolog->pushHandler(new \Monolog\Handler\ErrorLogHandler());
        });
    }
}
```

We also need to handle authentication (http basic auth in this case) so we'll create a simple middleware

```php
namespace App\Providers;
 
use Illuminate\Support\ServiceProvider;
 
class VcapServiceProvider extends ServiceProvider
{
    public function register()
    {
        if (php_sapi_name() == "cli-server") {
            $env = file_get_contents(__DIR__ . "/../../conf/vcap_services.json");
        } else {
            $env = $_ENV["VCAP_SERVICES"];
        }
 
        $vcapServices = json_decode($env, true);
 
        $this->app->bind('vcap_services', function ($app) use ($vcapServices) {
            return $vcapServices;
        });
    }
}
```

In summary: Lumen is cool. The interface is very similar to Silex. I can swap my mind from thinking in Silex to thinking in Lumen easily. Blade instead Twig: no problem. Service providers are very similar. Routing is almost the same and Middlewares are much better. Nowadays backend is a commodity for me so I don't want to spend to much time working on it. I want something that just work. Lumen looks like that.

Both projects: [Silex](https://github.com/gonzalo123/scp-cf-silex) and [Lumen](https://github.com/gonzalo123/scp-cf-lumen) are available in my github
