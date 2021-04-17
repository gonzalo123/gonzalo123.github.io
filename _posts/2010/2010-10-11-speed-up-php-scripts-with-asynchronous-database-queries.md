---
layout: post
title: "Speed up PHP scripts with asynchronous database queries"
date: "2010-10-11"
tags: 
  - "php"
categories: 
  - "php"
  - "postgreSQL"
---

That’s the situation. A web application with 4 heavy queries. Yes I know you can use a cache or things like that but imagine you must perform the four queries yes or yes. As I say before the four database queries are heavy ones. 2 seconds per one. Do it sequentially, that’s means our script will use at least 8 seconds (without adding the extra PHP time). Eight seconds are a huge time in a page load.

So here I will show up a technique to speed up our website using asynchronous calls.

One possible solution is fork each query into a separate thread but, does PHP support threads? The short is answer No. OK that’s not 100% true. We can use [PCNTL](http://php.net/manual/en/book.pcntl.php) functions but it’s a bit tricky for me to use them. I prefer to use this another solution to create fork and perform asynchronous operations.

The basic idea is isolate each query into a separate script.

```php
// fork.php
$dbh = new PDO('pgsql:dbname=pg1;host=localhost', 'nov', 'nov');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$t = filter_input(INPUT_GET, 't', FILTER_SANITIZE_STRING);
switch ($t) {
    case 0;
    case 1;
    case 2;
       $sql = "select * from test.tbl1";
       break;
}
$stmt = $dbh->prepare($sql);
$stmt->execute();
$data = $stmt->fetchAll();
echo json_encode($data);
```

And now we are going to use the [curl](http://php.net/manual/es/book.curl.php)'s multi functions to execute the database queries at the same time:

```php
// index.php
class Fork
{
    private $_handles = array();
    private $_mh      = array();
 
    function __construct()
    {
        $this->_mh = curl_multi_init();
    }
 
    function add($url)
    {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_HEADER, 0);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 30);
        curl_multi_add_handle($this->_mh, $ch);
        $this->_handles[] = $ch;
        return $this;
    }
 
    function run()
    {
        $running=null;
        do {
            curl_multi_exec($this->_mh, $running);
            usleep (250000);
        } while ($running > 0);
        for($i=0; $i < count($this->_handles); $i++) {
            $out = curl_multi_getcontent($this->_handles[$i]);
            $data[$i] = json_decode($out);
            curl_multi_remove_handle($this->_mh, $this->_handles[$i]);
        }
        curl_multi_close($this->_mh);
        return $data;
    }
}
 
$fork = new Fork;
 
$output = $fork->add("http://localhost/fork.php?t=0")
    ->add("http://localhost/fork.php?t=1")
    ->add("http://localhost/fork.php?t=2")
    ->run();
 
echo "<pre>";
print_r($output);
echo "</pre>";
```

And that’s all. Four queries will be executed at the same time and our scrip will use 2 seconds (the slowest query) instead of  8 seconds. But there's a little inconvenient within this solution. We need to open 4  connections instead of one because of the lack of connection pooling in PHP. Another problem with this technique appears if we need to debug our script. The debugger won’t debug our “forks” at the same time (we will need to debug then separately). Because of that we need to balance the usage of this technique or not, but it's another weapon we have to improve our scripts.
