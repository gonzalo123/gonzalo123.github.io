---
layout: post
title: "Playing with HTML5. Building a simple pool of WebWokers"
date: "2013-12-22"
tags: 
  - "html5"
  - "js"
  - "php"
  - "html5"
  - "pool"
  - "silex"
  - "webworker"
---

Today I'm playing with the HTML5's [WebWorkers](http://www.html5rocks.com/en/tutorials/workers/basics/). Since our JavaScript code runs within a single thread in our WebBrowser, heavy scripts can lock the execution of our code. HTML5 gives us one tool called WebWorkers to allow us to run different threads within our Application.

Today I'm playing with one small example (that's just an experiment). I want to create a pool of WebWebworkers and use them as a simple queue. The usage of the library is similar than the usage of WebWorkers. The following code shows how to start the pool with 10 instances of our worker "js/worker.js"

```javascript
// new worker pool with 10 instances of the worker
var pool = new WorkerPool('js/worker.js', 10);
 
// register callback to worker's onmessage event
pool.registerOnMessage(function (e) {
    console.log("Received (from worker): ", e.data);
});
```

"js/worker.js" is a standard WebWorker. In this example our worker perform XHR request to a API server (in this case one Silex application)

```javascript
importScripts('ajax.js');
 
self.addEventListener('message', function (e) {
    var data = e.data;
 
    switch (data.method) {
        case 'GET':
            getRequest(data.resource, function(xhr) {
                self.postMessage({status: xhr.status, responseText: xhr.responseText});
            });
            break;
    }
}, false);
```

WebWorkers runs in different scope than a traditional browser application. Not all JavaScript objects are available in the webworker scpope. For example we cannot access to "window" and DOM elements, but we can use XMLHttpRequest. In our experimente we're going to preform XHR requests from the webworker.

The library creates a queue with the variable number of workers:

```javascript
var WorkerPool;
 
WorkerPool = (function () {
    var pool = {};
    var poolIds = [];
 
    function WorkerPool(worker, numberOfWorkers) {
        this.worker = worker;
        this.numberOfWorkers = numberOfWorkers;
 
        for (var i = 0; i < this.numberOfWorkers; i++) {
            poolIds.push(i);
            var myWorker = new Worker(worker);
 
            +function (i) {
                myWorker.addEventListener('message', function (e) {
                    var data = e.data;
                    console.log("Worker #" + i + " finished. status: " + data.status);
                    pool[i].status = true;
                    poolIds.push(i);
                });
            }(i);
 
            pool[i] = {status: true, worker: myWorker};
        }
 
        this.getFreeWorkerId = function (callback) {
            if (poolIds.length > 0) {
                return callback(poolIds.pop());
            } else {
                var that = this;
                setTimeout(function () {
                    that.getFreeWorkerId(callback);
                }, 100);
            }
        }
    }
 
    WorkerPool.prototype.postMessage = function (data) {
        this.getFreeWorkerId(function (workerId) {
            pool[workerId].status = false;
            var worker = pool[workerId].worker;
            console.log("postMessage with worker #" + workerId);
            worker.postMessage(data);
        });
    };
 
    WorkerPool.prototype.registerOnMessage = function (callback) {
        for (var i = 0; i < this.numberOfWorkers; i++) {
            pool[i].worker.addEventListener('message', callback);
        }
    };
 
    WorkerPool.prototype.getFreeIds = function () {
        return poolIds;
    };
 
    return WorkerPool;
})();
```

The API server is a simple Silex application. This application also enables cross origin (CORS). You can read about it [here](http://gonzalo123.com/2013/12/16/enabling-cors-in-a-restfull-silex-server-working-with-a-phonegapcordova-application/ "Enabling CORS in a RESTFull Silex server, working with a phonegap/cordova applications").

```php
use Silex\Application;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
 
$app = new Application();
 
$app->get('/hello', function () use ($app) {
    error_log("GET /hello");
    sleep(2); // emulate slow process
    return $app->json(['method' => 'GET', 'response' => 'OK']);
});
 
$app->after(function (Request $request, Response $response) {
    $response->headers->set('Access-Control-Allow-Origin', '*');
});
```

You can see the whole code in my github [account](https://github.com/gonzalo123/workerpool).

Here one small screencast to see the application in action.

[![youtube](https://img.youtube.com/vi/HBynq8-3AaQ/0.jpg)](https://www.youtube.com/watch?v=HBynq8-3AaQ)
