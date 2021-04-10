---
layout: post
title: "Display errors on screen even with display errors = off with PHP"
date: "2011-10-10"
tags: 
  - "php"
  - "technology"
  - "tips"
  - "error_log"
---

Last month I was in a coding kata [session](http://katayunos.com/) playing with [StingCalculator](http://osherove.com/tdd-kata-1/), and between iteration and iteration we were taking about the problems of shared hostings. **Shared hosting** are cheap, but normally they don't allow us the use some kind of features. For example we **cannot see the error log**. That’s a problem when we need to see what happens within our application. Normally I work with my own servers, and I have got full access to error logs. But if we cannot see the error log and the server is configured with [display errors = off](http://es.php.net/manual/en/errorfunc.configuration.php#ini.display-errors) in php.ini (typical configuration in shared hosting), we have a problem.

One of my post came to my mind “[Real time monitoring PHP applications with websockets and node.js](http://gonzalo123.wordpress.com/2011/05/09/real-time-monitoring-php-applications-with-websockets-and-node-js/)”. Basically we can solve the problem using this technique, but if we are speaking about shared hosting without error log access, it’s very probably that we cannot install a node.js server or even use sockets functions. So it’s not “Real” solution to our problem.

The idea is create a variation of the original script (one kind of script’s Spin-off ;)). In this scprit we will capture errors and exceptions and show them in script at the end of the script. We don’t have access to the error log but we will show it in the browser.

Imagine the following script: 
```php
$a = 1/0;
throw new Exception("myException");
echo "hi";
```

It will show one warning (1/0) and one exception (myException)

- Warning: Division by zero
- Fatal error: Uncaught exception 'Exception' with message 'myException'

If we change php.ini show errors = off we will see a nice white screen.

The idea is add to the script: 

```php
include('ErrorSniffer.php');
ErrorSniffer::factory('127.0.0.1');
 
$a = 1/0;
throw new Exception("myException");
echo "hi";
```

Now with with [ErrorSniffer](https://github.com/gonzalo123/ErrorSniffer) library we will see a nice output, even with display errors = off. I also add a IP in the class constructor to restrict the output message to one IP. The idea is to place this script at production, so we don’t want to expose error messages to whole users.

If we see the source code of ErrorSniffer class we a constructor like this: \

```php
public function __construct($restingToIp)
{
    if ($this->getip() == $restingToIp) {
        self::register_exceptionHandler($this);
        self::set_error_handler($this);
        self::register_shutdown_function($this);
    }
}
 
private static function set_error_handler(ErrorSniffer &$that)
{
    set_error_handler(function ($errno, $errstr, $errfile, $errline) use (&$that) {
            $type = ErrorSniffer::getErrorName($errno);
            $that->registerError(array('type' => $type, 'message' => $errstr, 'file' => $errfile, 'line' => $errline));
            return false;
    });
}
 
private static function register_exceptionHandler(ErrorSniffer &$that)
{
    set_exception_handler(function($exception) use (&$that) {
            $exceptionName = get_class($exception);
            $message = $exception->getMessage();
            $file  = $exception->getFile();
            $line  = $exception->getLine();
            $trace = $exception->getTrace();
 
            $that->registerError(array('type' => 'EXCEPTION', 'exception' => $exceptionName, 'message' => $message, 'file' => $file, 'line' => $line, 'trace' => $trace));
            return false;
    });
}
 
private static function register_shutdown_function(ErrorSniffer &$that)
{
    register_shutdown_function(function() use (&$that) {
            $error = error_get_last();
 
            if ($error['type'] == E_ERROR) {
                $type = ErrorSniffer::getErrorName($error['type']);
                $that->registerError(array('type' => $type, 'message' => $error['message'], 'file' => $error['file'], 'line' => $error['line']));
            }
 
            $that->printErrors();
    });
}
```

As we can see we will catch errors and exceptions, we populate the member variable $errors and we also use [register\_shutdown\_function](http://php.net/manual/en/function.register-shutdown-function.php) to show information on shutdown.

You can see the full script at github [here](https://github.com/gonzalo123/ErrorSniffer).

What do you think?
