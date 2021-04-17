---
layout: post
title: "Building network services with PHP and xinetd"
date: "2010-05-23"
tags: 
  - "php"
categories: 
  - "php"
---

Not all is web and HTTP. Sometimes we need to create a network service listening to a port. We can create a TCP server in C, Java or even PHP but there's a really helpful daemon in Linux that helps us to do it. This daemon is xinetd. In this article we are going to create a network service with PHP and xinretd.

Now we are going to create our brand new service with xinetd and PHP. Let's start. First we are  going to create a simple network service listening to 60321 port. Our network service will say hello. The PHP script will be very complicated:

```php
// /home/gonzalo/tests/test1.php
echo "HELLO\n";
```

We want to create a network service on 60321 tcp port so we need to define this port on /etc/services. We put the following line at the end of /etc/services

```commandline
// /etc/services
...
myService   60321/tcp # my hello service
```

And finally we create out xinetd configuration script on the folder /etc/xinet.d/ , called myService (/etc/xinetd.d/myService)

```
# default: on
# description: my test service
 
service myService
{
        socket_type             = stream
        protocol                = tcp
        wait                    = no
        user                    = gonzalo
        server                  = /usr/local/bin/php-cli
        server_args             = /home/gonzalo/tests/test1.php
        log_on_success          += DURATION
        nice                    = 10
        disable                 = no
}
```

Now we restart xinetd

```commandline
sudo /etc/init.d/xinetd restart
```

And we have our network service ready:

```commandline
telnet localhost 60321
Trying ::1...
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
HELLO
Connection closed by foreign host.
```

Easy. isn't it? But it may be not really useful. So we are going to change something in our php script to accept input. Here we cannot use POST or GET parameters (that's not HTTP) so we need to read input from stdin. In PHP (and in other languajes too) that's pretty straightforward.

```php
$handle = fopen('php://stdin','r');
$input = fgets($handle);
fclose($handle);
 
echo "hello {$input}";
```

Now if we run our script from CLI it will ask for input. So if we test our network service with a telnet.

And we have our network service ready:

```commandline
telnet localhost 60321
Trying ::1...
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
```

we type: "gonzalo" and:

```commandline
gonzalo
hello gonzalo
Connection closed by foreign host.
```
