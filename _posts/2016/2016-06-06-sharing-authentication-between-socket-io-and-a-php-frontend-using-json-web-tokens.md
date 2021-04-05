---
layout: post
title: "Sharing authentication between socket.io and a PHP frontend (using JSON Web Tokens)"
date: "2016-06-06"
tags: 
  - "node-js"
  - "php"
  - "socket-io"
  - "technology"
  - "twig"
  - "websockets"
  - "jwt"
  - "silex"
  - "websockets"
---

I've written a previous post about [Sharing authentication between socket.io and a PHP frontend](http://gonzalo123.com/2016/05/16/sharing-authentication-between-socket-io-and-a-php-frontend/) but after publish the post a colleague (hi [@mariotux](https://twitter.com/mariotux)) told me that I can use JSON Web Tokens ([jwt](https://jwt.io/)) to do this. I had never used jwt before so I decided to study a little bit.

JWT are pretty straightforward. You only need to create the token and send it to the client. You don't need to store this token within a database. Client can decode and validate it on its own. You also can use any programming language to encode and decode tokens (jwt is available in the most common ones)

We're going to create the same example than the previous post. Today, with jwt, we don't need to pass the PHP session and perform a http request to validate it. We'll only pass the token. Our nodejs server will validate by its own. 

```javascript
var io = require('socket.io')(3000),
    jwt = require('jsonwebtoken'),
    secret = "my_super_secret_key";
 
// middleware to perform authorization
io.use(function (socket, next) {
    var token = socket.handshake.query.token,
        decodedToken;
    try {
        decodedToken = jwt.verify(token, secret);
        console.log("token valid for user", decodedToken.user);
        socket.connectedUser = decodedToken.user;
        next();
    } catch (err) {
        console.log(err);
        next(new Error("not valid token"));
        //socket.disconnect();
    }
});
 
io.on('connection', function (socket) {
    console.log('Connected! User: ', socket.connectedUser);
});
```

That's the client:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
Welcome {{ user }}!
 
<script src="http://localhost:3000/socket.io/socket.io.js"></script>
<script src="/assets/jquery/dist/jquery.js"></script>
 
<script>
    var socket;
    $(function () {
        $.getJSON("/getIoConnectionToken", function (jwt) {
            socket = io('http://localhost:3000', {
                query: 'token=' + jwt
            });
 
            socket.on('connect', function () {
                console.log("connected!");
            });
 
            socket.on('error', function (err) {
                console.log(err);
            });
        });
    });
</script>
 
</body>
</html>
```

And here the backend. A simple Silex server very similar than the previous post one. JWT has also several reserved [claims](https://jwt.io/introduction/). For example "exp" to set up an expiration timestamp. It's very useful. We only set one value and validator will reject tokens with incorrect timestamp. In this example I'm not using expiration date. That's means that my token will never expires. And never means never. In my first prototype I set up an small expiration date (10 seconds). That means my token is only available during 10 seconds. Sounds great. My backend generate tokens that are going to be used immediately. That's the normal situation but, what happens if I restart the socket.io server? The client will try to reconnect again using the token but it's expired. We'll need to create a new jwt before reconnecting. Because of that I've removed expiration date in this example but remember: Without expiration date your generated tokens will be always valid (al always is a very big period of time)

```php
<?php
include __DIR__ . "/../vendor/autoload.php";
 
use Firebase\JWT\JWT;
use Silex\Application;
use Silex\Provider\SessionServiceProvider;
use Silex\Provider\TwigServiceProvider;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;
 
$app = new Application([
    'secret' => "my_super_secret_key",
    'debug' => true
]);
$app->register(new SessionServiceProvider());
$app->register(new TwigServiceProvider(), [
    'twig.path' => __DIR__ . '/../views',
]);
 
$app->get('/', function (Application $app) {
    return $app['twig']->render('home.twig');
});
$app->get('/login', function (Application $app) {
    $username = $app['request']->server->get('PHP_AUTH_USER', false);
    $password = $app['request']->server->get('PHP_AUTH_PW');
    if ('gonzalo' === $username && 'password' === $password) {
        $app['session']->set('user', ['username' => $username]);
 
        return $app->redirect('/private');
    }
    $response = new Response();
    $response->headers->set('WWW-Authenticate', sprintf('Basic realm="%s"', 'site_login'));
    $response->setStatusCode(401, 'Please sign in.');
 
    return $response;
});
 
$app->get('/getIoConnectionToken', function (Application $app) {
    $user = $app['session']->get('user');
    if (null === $user) {
        throw new AccessDeniedHttpException('Access Denied');
    }
 
    $jwt = JWT::encode([
        // I can use "exp" reserved claim. It's cool. My connection token is only available
        // during a period of time. The problem is if I restart the io server. Client will
        // try to re-connect using this token and it's expired.
        //"exp"  => (new \DateTimeImmutable())->modify('+10 second')->getTimestamp(),
        "user" => $user
    ], $app['secret']);
 
    return $app->json($jwt);
});
 
$app->get('/private', function (Application $app) {
    $user = $app['session']->get('user');
 
    if (null === $user) {
        throw new AccessDeniedHttpException('Access Denied');
    }
 
    $userName = $user['username'];
 
    return $app['twig']->render('private.twig', [
        'user'  => $userName
    ]);
});
$app->run();
```

Full project in my [github](https://github.com/gonzalo123/ioauthjwt).
