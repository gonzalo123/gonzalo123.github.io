---
title: "Building a REST client with asynchronous calls using PHP and curl"
date: "2010-02-06"
tags: 
  - "php"
categories: 
  - "php"
---

One month ago I posted an article called Building a simple HTTP client with PHP. A REST client. In this post I tried to create a simple fluid interface to call REST webservices in PHP. Some days ago I read an [article](http://www.ibuildings.nl/blog/archives/811-Multithreading-in-PHP-with-CURL.html) and one light switch on on my mind. I can use curl's "multi" functions to improve my library and perform simultaneous calls very easily.

I've got a project that needs to call to different webservices. Those webservices sometimes are slow (2-3 seconds). If I need to call to, for example, three webservices my script will use the add of every single call's time. With this improve to the library I will use only the time of the slowest webservice. 2 seconds instead of 2+2+2 seconds. Great.

For the example I've created a really complex php script that sleeps x seconds depend on an input param:

```php
sleep((integer) $_REQUEST['sleep']);
echo $_REQUEST['sleep'];
```

With synchronous calls:

```php
echo Http::connect('localhost', 8082)
    ->doGet('/tests/gam_http/sleep.php', array('sleep' => 3));
echo Http::connect('localhost', 8082)
    ->doPost('/tests/gam_http/sleep.php', array('sleep' => 2));
echo Http::connect('localhost', 8082)
    ->doGet('/tests/gam_http/sleep.php', array('sleep' => 1));
```

This script takes more or less 6 seconds (3+2+1)

But If I switch it to:

```php
$out = Http::connect('localhost', 8082)
    ->get('/tests/gam_http/sleep.php', array('sleep' => 3))
    ->post('/tests/gam_http/sleep.php', array('sleep' => 2))
    ->get('/tests/gam_http/sleep.php', array('sleep' => 1))
    ->run();
print_r($out);
```

The script only uses 3 seconds (the slowest process)

I've got a project that uses it. But I have a problem. I have webservices in different hosts so I've done a bit change to the library:

```php
$out = Http::multiConnect()
    ->add(Http::connect('localhost', 8082)->get('/tests/gam_http/sleep.php', array('sleep' => 3)))
    ->add(Http::connect('localhost', 8082)->post('/tests/gam_http/sleep.php', array('sleep' => 2)))
    ->add(Http::connect('localhost', 8082)->get('/tests/gam_http/sleep.php', array('sleep' => 1)))
    ->run();
```

With a single connection, the exceptions are easy to implement. If curl\_getinfo() returns an error message I throw an exception, but now with a multiple interface how I can do it? I throw an exception if one call fail, or not? I have decided not to use exceptions in multiple interface. I always return an array with all the output of every webservice's call and if something wrong happens instead of the output I will return an instance of Http\_Multiple\_Error class. Why I use a class instead of a error message? The answer is easy. If I want to check all the answers I can check if any of them is an instanceof Http\_Multiple\_Error. Also I don't want to check anything I put a silentMode() function to switch off all error messages.

```php
$out = Http::multiConnect()
    ->silentMode()
    ->add(Http::connect('localhost', 8082)->get('/tests/gam_http/sleep.php', array('sleep' => 3)))
    ->add(Http::connect('localhost', 8082)->post('/tests/gam_http/sleep.php', array('sleep' => 2)))
    ->add(Htt
```

The full code is available on [google code](http://code.google.com/p/gam-http/) but the main function is the following one:

```php
...
private function _run()
{
    $headers = $this->_headers;
    $curly = $result = array();
 
    $mh = curl_multi_init();
    foreach ($this->_requests as $id => $reg) {
        $curly[$id] = curl_init();
 
        $type     = $reg[0];
        $url       = $reg[1];
        $params = $reg[2];
 
        if(!is_null($this->_user)){
           curl_setopt($curly[$id], CURLOPT_USERPWD, $this->_user.':'.$this->_pass);
        }
 
        switch ($type) {
            case self::DELETE:
                curl_setopt($curly[$id], CURLOPT_URL, $url . '?' . http_build_query($params));
                curl_setopt($curly[$id], CURLOPT_CUSTOMREQUEST, self::DELETE);
                break;
            case self::POST:
                curl_setopt($curly[$id], CURLOPT_URL, $url);
                curl_setopt($curly[$id], CURLOPT_POST, true);
                curl_setopt($curly[$id], CURLOPT_POSTFIELDS, $params);
                break;
            case self::GET:
                curl_setopt($curly[$id], CURLOPT_URL, $url . '?' . http_build_query($params));
                break;
        }
        curl_setopt($curly[$id], CURLOPT_RETURNTRANSFER, true);
        curl_setopt($curly[$id], CURLOPT_HTTPHEADER, $headers);
 
        curl_multi_add_handle($mh, $curly[$id]);
    }
 
    $running = null;
    do {
        curl_multi_exec($mh, $running);
        sleep(0.2);
    } while($running > 0);
 
    foreach($curly as $id => $c) {
        $status = curl_getinfo($c, CURLINFO_HTTP_CODE);
        switch ($status) {
            case self::HTTP_OK:
            case self::HTTP_CREATED:
            case self::HTTP_ACEPTED:
                $result[$id] = curl_multi_getcontent($c);
                break;
            default:
                if (!$this->_silentMode) {
                    $result[$id] = new Http_Multiple_Error($status, $type, $url, $params);
                }
        }
        curl_multi_remove_handle($mh, $c);
    }
 
    curl_multi_close($mh);
    return $result;
```
