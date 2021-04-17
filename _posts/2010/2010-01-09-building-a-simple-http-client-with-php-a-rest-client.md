---
layout: post
title: "Building a simple HTTP client with PHP. A REST client"
date: "2010-01-09"
tags: 
  - "php"
categories: 
  - "php"
---

REST webservices are cool. Nowadays there are a lot of APIS with REST and in combination with JSONP is really easy to build applications. Actually I am playing with [CouchDB](http://couchdb.apache.org/). It has a nice RESTfull API and I want to use it with PHP. There are several PHP libraries, even a nice PHP extension to work with CouchDB. I like Zend Framework's REST [client](http://framework.zend.com/manual/en/zend.rest.client.html) but as exercise I will develop a HTTP client. My idea is to create a simple class that allows me to perform GET, POST and DELETE requests to a remote server.

That's the way I want to use my class:

```php
echo Http::connect('localhost', 8082)
    ->doGet('tests/httpclient/dummy.php', array('a' => "a a a"));
```

First I want to do a lazy connection to server. That's because I want to create the real connection only when I create the request. So I create a factory function that only saves the configuration of the client:

```php
private $_host = null;
private $_port = null;
private $_user = null;
private $_pass = null;
 
/**
 * Factory of the class. Lazy connect
 *
 * @param string $host
 * @param integer $port
 * @param string $user
 * @param string $pass
 * @return Http
 */
static public function connect($host, $port, $user=null, $pass=null)
{
    return new self($host, $port, $user, $pass);
}
```

And now one public function for each action:

```php
const POST   = 'POST';
const GET    = 'GET';
const DELETE = 'DELETE';
 
/**
 * POST request
 *
 * @param string $url
 * @param array $params
 * @return string
 */
public function doPost($url, $params=array())
{
    return $this->_exec(self::POST, $this->_url($url), $params);
}
 
/**
 * GET Request
 *
 * @param string $url
 * @param array $params
 * @return string
 */
public function doGet($url, $params=array())
{
    return $this->_exec(self::GET, $this->_url($url), $params);
}
 
/**
 * DELETE Request
 *
 * @param string $url
 * @param array $params
 * @return string
 */
public function doDelete($url, $params=array())
{
    return $this->_exec(self::DELETE, $this->_url($url), $params);
}
```

And one extra function to set the headers of the request

```php
private $_headers = array();
/**
 * setHeaders
 *
 * @param array $headers
 * @return Http
 */
public function setHeaders($headers)
{
    $this->_headers = $headers;
    return $this;
}
```

I use a settler function and chainable (returns the instance of the class) because if I need to use some custom headers I want to use the following interface:

```php
echo Http::connect('localhost', 8082)
    ->setHeaders($myCustomHeaders)
    ->doGet('tests/httpclient/dummy.php', array('a' => "a a a"));
```

And finally the main function. The function that perform the real request. There are several ways to do that in PHP. For this example I will use CURL's functions. You must take care about it because CURL extension is not always available in PHP installations. So please check your phpinfo(). If CURL is not available and if you are not allowed to install it you must chose another alternatives.

```php
const HTTP_OK = 200;
const HTTP_CREATED = 201;
const HTTP_ACEPTED = 202;
 
/**
 * Performing the real request
 *
 * @param string $type
 * @param string $url
 * @param array $params
 * @return string
 */
private function _exec($type, $url, $params = array())
{
    $headers = $this->_headers;
    $s = curl_init();
 
    if(!is_null($this->_user)){
       curl_setopt($s, CURLOPT_USERPWD, $this->_user.':'.$this->_pass);
    }
 
    switch ($type) {
        case self::DELETE:
            curl_setopt($s, CURLOPT_URL, $url . '?' . http_build_query($params));
            curl_setopt($s, CURLOPT_CUSTOMREQUEST, self::DELETE);
            break;
        case self::POST:
            curl_setopt($s, CURLOPT_URL, $url);
            curl_setopt($s, CURLOPT_POST, true);
            curl_setopt($s, CURLOPT_POSTFIELDS, $params);
            break;
        case self::GET:
            curl_setopt($s, CURLOPT_URL, $url . '?' . http_build_query($params));
            break;
    }
 
    curl_setopt($s, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($s, CURLOPT_HTTPHEADER, $headers);
    $_out = curl_exec($s);
    $status = curl_getinfo($s, CURLINFO_HTTP_CODE);
    curl_close($s);
    switch ($status) {
        case self::HTTP_OK:
        case self::HTTP_CREATED:
        case self::HTTP_ACEPTED:
            $out = $_out;
     &nbsp;      break;
        default:
            throw new Http_Exception("http error: {$status}", $status);
    }
    return $out;
}
```

I've have also created a Http\_Exception Exception class:

```php
class Http_Exception extends Exception{
    const NOT_MODIFIED = 304;
    const BAD_REQUEST = 400;
    const NOT_FOUND = 404;
    const NOT_ALOWED = 405;
    const CONFLICT = 409;
    const PRECONDITION_FAILED = 412;
    const INTERNAL_ERROR = 500;
}
```

Http throws Http\_Exception when something wrong happens so I can play with it to build my application logic. For example:

```php
try {
   echo Http::connect('localhost', 8082)
        ->doGet('tests/couchdb/a.php', array('a' => "a a a"));
} catch (Http_Exception $e) {
    switch ($e) {
        case Http_Exception::INTERNAL_ERROR:
            // do something
            break;
    }
}
```

And that's it. There is the full code available on [bitbucket](http://bitbucket.org/gonzalo123/gam_http/).

**UPDATED**. See a variation of this class in the next article: [Building a REST client with asynchronous calls using PHP and curl](http://gonzalo123.wordpress.com/2010/02/06/building-a-rest-client-with-asynchronous-calls-using-php-and-curl/)
