---
layout: post
title: "Coding Katas, TDD and Katayunos"
date: "2011-12-12"
tags: 
  - "node-js"
  - "php"
  - "technology"
---

2011 is about to finish, and I want to speak about my way through the world of TDD. In the beginning of the year appeared a new cool project called [12meses12katas](http://12meses12katas.com/) (12 months - 12 katas). The aim of the project was to propose one coding kata per month, and allow to the people to publish their implementation of the kata over [github](https://github.com/12meses12katas). In the line of this project a crew of crazy [coders](https://twitter.com/#!/programania/katayuners/members) started another cool project called [Katayunos](http://katayunos.com/). Katayunos is a small pun with the word Desayuno (Breakfast) and Coding Kata. It's something similar than a code retreat. The purpose of the katayunos was to meet together in one place one saturday morning, have a breakfast and play with pair programming and TDD with one coding kata. Something too geek to explain to non-geek people, I known, but if you are reading this, It's probable that your understand this ;). Our first katayuno was in the cafeteria of one Hotel. One cold Saturday morning (a really cold [one](http://www.twitpic.com/3say15) believe me). The main problem was that we didn't have any electrical plug, so our pomodoros were marked by the laptop batteries.

After this cold first katayuno we achieve new places with wifi and those kind of luxuries. Those kind of events are really interesting because we meet together different people and different programming languages. I still remember when I played with c# one time (I still have the sulfur smell in my fingers :) ).

Basically the idea is:

- Choose a coding kata
- Pomodoros of 25 minutes
- Pair programming (change the couples with each iteration)
- Have fun

And now I'm going to show my 12 implementations of the 12 katas from 12meses12katas's project:

**Jaunary: [String-Calculator](http://osherove.com/tdd-kata-1/)** A good kata for beginners. That's a good one because the testing strategy is very clear and we can focus on Testing: Red (test NOK) -> implement code -> Green (test OK). My solution: [php + phpunit](https://github.com/gonzalo123/Enero-String-Calculator/tree/master/gonzalo123)

**February: [Roman-Numerals](http://codingdojo.org/cgi-bin/wiki.pl?KataRomanNumerals)** I really hate roman numerals after doing this kata ;) My solution: [python + pyunit](https://github.com/gonzalo123/Febrero-Roman-Numerals/tree/master/gonzalo123)

**March: [FizzBuzz](http://www.codingdojo.org/cgi-bin/wiki.pl?KataFizzBuzz)** A famous kata. A Really simple one. Because of that we can focus on the TDD (baby steps, refactoring, ...) My solution: [php + phpunit](https://github.com/gonzalo123/Marzo-FizzBuzz/tree/master/gonzalo123) I also wrote a [post](http://kcy.me/65dp) with a crazy dynamic classes and the implementation of FizzBuzz kata with those dinamic classes:

**April: [Bowling](http://codingdojo.org/cgi-bin/wiki.pl?KataBowling)** My problem was that I didn't know the rules of Bowling so I needed to learn how to play Bowling first. My solution: [php + phpunit](https://github.com/gonzalo123/Abril-Bowling/tree/master/gonzalo123)

**May: [KataLonja](http://agilismo.es/codekatas/109-katalonja)** A strange kata. I like it because it's not a theoretical one. It's close to a real problem. My solution: [php + phpunit](https://github.com/gonzalo123/Mayo-KataLonja/tree/master/gonzalo123)

**June: [Simple-Lists](http://codekata.pragprog.com/2007/01/kata_twenty_one.html)** It was my first kata with javascript and node.js. My solution: [node.js + nodeunit](https://github.com/gonzalo123/Junio-Simple-Lists/tree/master/gonzalo123) (with John Resig's class implementation)

**July: [Prime-Factors](http://butunclebob.com/ArticleS.UncleBob.ThePrimeFactorsKata)** Another famous one: Prime factors. My solution: [node.js + nodeunit](https://github.com/gonzalo123/Julio-Prime-Factors/tree/master/gonzalo123)

**August: [Minesweeper](http://codingdojo.org/cgi-bin/wiki.pl?KataMinesweeper)** An implementation of the famous computer game Minesweeper. My solution: [php + phpunit](https://github.com/gonzalo123/Agosto-Minesweeper/tree/master/gonzalo123)

**October: [Karate-Chop](http://codekata.pragprog.com/2007/01/kata_two_karate.html)** Dual solution. With php and JavaScript (with node.js) My solution: [php + phpunit and node.js + nodeunit](https://github.com/gonzalo123/Octubre-Karate-Chop/tree/master/gonzalo123)

**November: [KataPotter](http://codingdojo.org/cgi-bin/wiki.pl?KataPotter)** This time I will use coffeescript instead of pure js My solution: [node.js + coffeeScript + jasmine](https://github.com/gonzalo123/Noviembre-KataPotter/tree/master/gonzalo123)

**December: [PomodoroKata](http://agilismo.es/2011/12/06/pomodorokata/)** CoffeeScript again. One pomodoro is one pomodoro. Let's build a pomodoro timer in a pomodoro. My solution: [node.js + coffeeScript + jasmine](https://github.com/gonzalo123/Diciembre-PomodoroKata/tree/master/gonzalo123)

So, what are you waiting for to create a local group and clone the katayuno in your city? It's very easy: broadcast the event over twitter, pick a place, pick a kata [and](http://www.flickr.com/photos/ggalmazor/sets/72157626032025541/) [enjoy](http://www.flickr.com/photos/joserra/sets/72157626021532617/).

Regards, [Gonzalo](https://twitter.com/#!/gonzalo123)
