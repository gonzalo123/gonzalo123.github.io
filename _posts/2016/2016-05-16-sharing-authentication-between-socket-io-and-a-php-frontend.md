---
layout: post
title: "Sharing authentication between socket.io and a PHP frontend"
date: "2016-05-16"
tags: 
  - "node-js"
  - "php"
  - "technology"
  - "websockets"
  - "js"
  - "nodejs"
  - "sessions"
  - "silex"
  - "socket-io"
---

Normally, when I work with [websockets](https://gonzalo123.com/?s=websockets), my stack is a [socket.io](http://socket.io/) server and a [Silex](http://silex.sensiolabs.org/) frontend. Protect a PHP frontend with one kind of authentication of another is pretty straightforward. But if we want to use websockets, we need to set up another server and if we protect our frontend we need to protect our websocket server too.

If our frontend is node too ([express](http://expressjs.com/) for example), sharing authentication is more easy but at this time we we want to use two different servers (a node server and a PHP server). I've written about [it](https://gonzalo123.com/2013/12/24/integrating-websockets-with-php-applications-silex-and-socket-io-playing-together/) too but today we\`ll see another solution. Let's start.

Imagine we have this simple Silex application. It has three routes:

- "/" a public route
- "/login" to perform the login action
- "/private" a private route. If we try to get here without a valid session we'll get a 403 error

And this is the code. It's basically one example using sessions taken from Silex [documentation](http://silex.sensiolabs.org/doc/providers/session.html):

\[sourcecode language="php"\] use Silex\\Application; use Silex\\Provider\\SessionServiceProvider; use Silex\\Provider\\TwigServiceProvider; use Symfony\\Component\\HttpFoundation\\Response; use Symfony\\Component\\HttpKernel\\Exception\\AccessDeniedHttpException;

$app = new Application();

$app->register(new SessionServiceProvider()); $app->register(new TwigServiceProvider(), \[ 'twig.path' => \_\_DIR\_\_ . '/../views', \]);

$app->get('/', function (Application $app) { return $app\['twig'\]->render('home.twig'); });

$app->get('/login', function () use ($app) { $username = $app\['request'\]->server->get('PHP\_AUTH\_USER', false); $password = $app\['request'\]->server->get('PHP\_AUTH\_PW');

if ('gonzalo' === $username && 'password' === $password) { $app\['session'\]->set('user', \['username' => $username\]);

return $app->redirect('/private'); }

$response = new Response(); $response->headers->set('WWW-Authenticate', sprintf('Basic realm="%s"', 'site\_login')); $response->setStatusCode(401, 'Please sign in.');

return $response; });

$app->get('/private', function () use ($app) { $user = $app\['session'\]->get('user'); if (null === $user) { throw new AccessDeniedHttpException('Access Denied'); }

return $app\['twig'\]->render('private.twig', \[ 'username' => $user\['username'\] \]); });

$app->run(); \[/sourcecode\]

Our "/private" route also creates a connection with our websocket server.

\[sourcecode language="html"\] <!DOCTYPE html> <html lang="en"> <head> <meta charset="UTF-8"> <title>Title</title> </head> <body> Welcome {{ username }}!

<script src="http://localhost:3000/socket.io/socket.io.js"></script> <script> var socket = io('http://localhost:3000/'); socket.on('connect', function () { console.log("connected!"); }); socket.on('disconnect', function () { console.log("disconnected!"); }); </script>

</body> </html> \[/sourcecode\]

And that's our socket.io server. A really simple one.

\[sourcecode language="js"\] var io = require('socket.io')(3000); \[/sourcecode\]

It works. Our frontend is protected. We need to login with our credentials (in this example "gonzalo/password"), but everyone can connect to our socket.io server. The idea is to use our PHP session to protect our socket.io server too. In fact is very easy how to do it. First we need to pass our PHPSESSID to our socket.io server. To do it, when we perform our socket.io connection in the frontend, we pass our session id \[sourcecode language="html"\] <script> var socket = io('http://localhost:3000/', { query: 'token={{ sessionId }}' }); socket.on('connect', function () { console.log("connected!"); }); socket.on('disconnect', function () { console.log("disconnect!"); }); </script> \[/sourcecode\]

As well as we're using a twig template we need to pass sessionId variable

\[sourcecode language="php"\] $app->get('/private', function () use ($app) { $user = $app\['session'\]->get('user'); if (null === $user) { throw new AccessDeniedHttpException('Access Denied'); }

return $app\['twig'\]->render('private.twig', \[ 'username' => $user\['username'\], 'sessionId' => $app\['session'\]->getId() \]); }); \[/sourcecode\]

Now we only need to validate the token before stabilising connection. Socket.io provides us a [middleware](http://socket.io/docs/server-api/#namespace#use(fn:function):namespace) to perform those kind of operations. In this example we're using PHP sessions out of the box. How can we validate it? The answer is easy. We only need to create a http client (in the socket.io server) and perform a request to a protected route (we'll use "/private"). If we're using a different provider to store our sessions (I [hope](http://paul-m-jones.com/archives/2375) you aren't using Memcached to store PHP session, indeed) you'll need to validate our sessionId against your provider.

\[sourcecode language="js"\] var io = require('socket.io')(3000), http = require('http');

io.use(function (socket, next) { var options = { host: 'localhost', port: 8080, path: '/private', headers: {Cookie: 'PHPSESSID=' + socket.handshake.query.token} };

http.request(options, function (response) { response.on('error', function () { next(new Error("not authorized")); }).on('data', function () { next(); }); }).end(); });

io.on('connection', function () { console.log("connected!"); }); \[/sourcecode\]

Ok. This example works but we're generating dynamically a js file injecting our PHPSESSID. If we want to extract the sessionId from the request we can use document.cookie but sometimes it doesn't work. That's because [HttpOnly](http://blog.codinghorror.com/protecting-your-cookies-httponly/). HttpOnly is our friend if we want to protect our cookies against XSS attacks but in this case our protection difficults our task.

We can solve this problem performing a simple request to our server. We'll create a new route (a private route) called 'getSessionID' that gives us our sessionId. \[sourcecode language="php"\] $app->get('/getSessionID', function (Application $app) { $user = $app\['session'\]->get('user'); if (null === $user) { throw new AccessDeniedHttpException('Access Denied'); }

return $app->json($app\['session'\]->getId()); }); \[/sourcecode\]

So before establishing the websocket we just need to create a GET request to our new route to obtain the sessionID.

\[sourcecode language="js"\] var io = require('socket.io')(3000), http = require('http');

io.use(function (socket, next) { var sessionId = socket.handshake.query.token, options = { host: 'localhost', port: 8080, path: '/getSessionID', headers: {Cookie: 'PHPSESSID=' + sessionId} };

http.request(options, function (response) { response.on('error', function () { next(new Error("not authorized")); }); response.on('data', function (chunk) { var sessionIdFromRequest; try { sessionIdFromRequest = JSON.parse(chunk.toString()); } catch (e) { next(new Error("not authorized")); }

if (sessionId == sessionIdFromRequest) { next(); } else { next(new Error("not authorized")); } }); }).end(); });

io.on('connection', function (socket) { setInterval(function() { socket.emit('hello', {hello: 'world'}); }, 1000); }); \[/sourcecode\]

And thats all. You can see the full example in my [github](https://github.com/gonzalo123/ioauth) account.
