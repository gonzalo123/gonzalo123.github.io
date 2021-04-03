---
layout: post
title: "Silex is dead (... or not)"
date: "2017-07-10"
categories: 
  - "lumen"
  - "php"
  - "symfony"
  - "technology"
tags: 
  - "desymfony"
  - "laravel"
  - "silex"
coverImage: "112871.png"
---

The last week was [deSymfony](http://desymfony.com/) conference in Castellón (Spain). IMHO deSymfony is the best conference I've ever attended. The talks are good but from time to now I appreciate this kind of events not because of them. I like to go to events because of people, the coffee breaks and the community (and in deSymfony is brilliant at this point). This year I cannot join to the conference. It was a pity. A lot of good friends there. So I only can follow the buzz in Twitter, read the published [slides](https://gist.github.com/raulfraile/0d80096f9b442df46d92e85886be8cba) (thanks [Raul](https://twitter.com/raulfraile)) and wait for the talk videos in [youtube](https://www.youtube.com/user/desymfony).

In my Twitter timeline especially two tweets get my attention. One tweet was from [Julieta Cuadrado](https://twitter.com/julietacuadrado) and another one from [Asier Marqués](https://twitter.com/asiermarques). 

![](/assets/images/silex_esta_muerto_-_bc3basqueda_en_twitter.png)

Tweets are in Spanish but the translation is clear: [Javier Eguiluz](https://github.com/javiereguiluz) (Symfony Core Team member and co-organizer of the conference) said in his talk: “**Silex is dead**”. At the time I read the tweets his slides were not available yet, but a couple of days after the slides were [online](https://www.slideshare.net/javier.eguiluz/desymfony-2017-symfony-4-symfony-flex-y-el-futuro-de-symfony). The slide 175 is clear “Silex is dead”

![](/assets/images/desymfony_2017__symfony_4__symfony_flex_y_el_futuro_de_symfony.png)


Javier recommends us not to use [Silex](https://silex.sensiolabs.org/) in future new projects and mark existing ones as “_legacy_”. It's hard to me. If you have ever read my blog you will notice that I'm a big fan of [Silex](https://gonzalo123.com/tag/silex/). Each time I need a backend, a API/REST server of something like that the first thing I do is “composer require silex/silex”. I know that Silex has limitations. It's built on top of Pimple dependency injection container and Pimple is really awful, but this microframework gives to me exactly what I need. It's small, simple, fast enough and really easy to adapt to my needs.

I remember a dinner in deSymfony years ago speaking with Javier in Barcelona. He was trying to "convince" me to use Symfony full stack framework instead of Silex. He almost succeeded, but I didn't like Symfony full stack. Too complicated for me. A lot interesting things but a lot of them I don't really need. I don't like Symfony full stack framework, but I love Symfony. For me it's great because of its components. They're independent pieces of code that I can use to fit exactly to my needs instead of using a full-stack framework. I've learn a lot SOLID reading and hacking with Symfony components. I'm not saying that full stack frameworks are bad. I only say that they're not for me. If I'm forced to use them I will do it, but if I can choose, I definitely choose a micro framework, even for medium an big projects.

New version of Symfony (Symfony 4) is coming next November and reading the slides of Javier at slideshare I can get an idea of its roadmap. My summary is clear: “Brilliant”. It looks like the people of Symfony listen to my needs and change all the framework to adapt it to me. After understand the roadmap I think that I need to change to title of the post (Initially it was only "Silex is dead"). Silex is not dead. For me Symfony (the full stack framework) is the death. Silex will be upgraded and will be renamed to Symfony (I know that this assertion is subjective. It's just my point of view). So the bad feeling that I felt when I read Julieta and Asier's tweets turns into a good one. Good move SensioLabs!

But I've got a problem right now. What can I do if I need to start a new project today? Symfony 4 isn't ready yet. Javier said that we can use [Symfony Flex](https://github.com/symfony/flex) and create new projects with Symfony 3 with the look and feel of Symfony 4, but Flex is still in alpha and I don't want to play with alpha tools in production. Especially in the backend. I'm getting older, I know. For me the backend is a commodity right now. I need the backend to serve JSON mainly.

I normally use PHP and Silex here only because I'm very confortable with it. In the projects, business people doesn't care about technologies and frameworks. It's our job (or our problem depending on how to read it). And don't forget one thing: Developers are part of business, so in one part of my mind I don't care about frameworks also. I care about making things done, maximising the potential of technology and driving innovation to customer benefits (good lapidary phrase, isn't it?).

So I've started looking for alternatives. My objective here is clear: I want to find a framework to do the things that I usually do with Silex. Nothing more. And there's something important here: The tool must be easy to learn. I want to master (or at least become productive) the tool in a couple of days maximum.

I started with the first one: [Lumen](https://lumen.laravel.com/) and I think I will stop searching. Lumen is the micro framework of [Laravel](https://laravel.com/). Probably in the PHP world now there're two major communities: Symfony and Laravel. Maybe if we're strict Laravel and Symfony are not different communities. In fact Laravel and Symfony shares a lot of components. So maybe both communities are the same.

I've almost never played with Laravel and it's time to study it a little bit. Time ago I used Eloquent ORM but since I hate ORMs I always return to PDO/DBAL. As I said before I didn't like Symfony full stack framework. It's too complex for me, and Laravel is the same. When I started with PHP (in the early 2000) there weren't any framework. I remember me reading books of Java and J2EE. Trying to understand something in its nightmare of acronyms, XMLs configurations and trying to build my own framework in PHP. Now in 2017 to build our own framework in PHP is good learning point but use it with real projects is ridiculous. As someone said before (I don't remember who) "_Everybody must build his own framework, and never use it at all_".

Swap from Silex to Lumen is pretty straightforward. In fact with one ultra-minimal application it's exactly the same:

```php
use Silex\Application;
 
$app = new Application();
 
$app->get("/", function() {
    return "Hello from Silex";
});
 
$app->run();
```

```php
use Laravel\Lumen\Application;
 
$app = new Application();
 
$app->get("/", function() {
    return "Hello from Lumen";
});
 
$app->run();
```

If you're a Silex user you only need a couple of hours reading the Lumen [docs](https://lumen.laravel.com/docs) and you will be able to set up a new project without any problem. Concepts are the same, slight differences and even cool things such as groups and middlewares. Nothing impossible to do with Silex, indeed, but with a very smart and simple interface. If I need to create a new project right now I will use Lumen without any doubt.

Next winter, when Symfony 4 arrives, I probably will face the problem of choose. But past years I've been involved into the crazy world of JavaScript: Angular, Angular2, React, npm, yarn, webpack, … If I've survived this (finally I choose JQuery, but that's a different story :), I am ready for all right now.
