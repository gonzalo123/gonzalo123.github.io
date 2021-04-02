---
layout: post
permalink: /:year/:month/:day/:title/
title: "Should we use our own frameworks in a production system?"
date: "2009-12-07"
categories: 
  - "php"
  - "web-development"
---

Some days ago I read an article called [why every developer should write their own framework](http://www.brandonsavage.net/why-every-developer-should-write-their-own-framework/ "why every developer should write their own framework"). It's an interesting post and everybody are agree with the author. Build a framework is a good way to learn other frameworks. You face problems and you give solutions to those problems similar than other framework's ones. It's also good see the code of other frameworks to see the differences. Definitely I am fully agree with Brandon.

But my question is more complicated. Should we use our own frameworks in a production system? I think the answer is not as clear as the fact of building or own framework. I will try to answer (or at least give my answer) in two scenarios. One as a developer and another one as an IT Manager.

**Answer for a developer**

As a developer is a difficult answer. We are agree build our own framework is a good exercise but use it in a production system?. When you build your own framework you are the biggest expert in your framework. You don't need to read books to learn your framework. You need to read books to take ideas and learn more things from another ones but not on your framework. That's a good point. But you must code a lot. You must code solutions that other frameworks has done already implemented. So your own framework speeds up the learning curve (when the framework is done or at least done) but slows down the developing time when you need to code core functions. If your framework is finished and it works under all situations and all possible future possibilities had been taken into account, the use of your own framework is the best idea. But your own framework will be never finished. It will be always under construction so the answer is not easy. I also don't like to develop solutions for non-existing problems. I prefer to develop the solution when the problem appear. I like think in possible problems and think in a possible solution but not code the solution until the problem appear. Maybe I can can use a sandbox to develop some ideas but not add them to the framework. so at least for me my frameworks are never finished.

As a developer another question appear. Does my framework improve my resume. I'm not a person who learn things only to improve my resume but, like or not, resume is important. If you are an expert in [Zend](http://framework.zend.com/), [Cake](http://cakephp.org/) or [Symfony](http://www.symfony-project.org/) you will have maybe more possibilities getting a job. It will be very difficult to find a job offer looking for an expert in your own framework. isn't it? But If you build your own framework and if you are in touch with the frameworks of the market (you read articles, books, and you play a bit with them) it will be very easy to adapt yourself to another exiting framework.

**Answer for an IT Manager**

If you are an IT Manager and you need to choose between custom-made framework and existing one I will give you two possible answer. the sort one and the long one. The sort one is use an existing one. If you need to build an IT team it will be easy to hire developers if you chose an existing framework. [Zend](http://framework.zend.com/), [Cake](http://cakephp.org/), [Symfony](http://www.symfony-project.org/) are good frameworks. They are open source and is easy find developers. Of course you must chose only open source frameworks. They are good enough, widely used and free. Free as free beer. Something very important nowadays.

Now the long answer: Depend on your team. If you don't have a team and you need to hire use the sort answer and use an existing one. But if you have a team the answer is not as easy as if you don't have it. Home made framework means the knowledge is in-house. Home made framework means you don't need to paid any external consultant to solve anything or to add any new functionality. But home-made framework means you cannot do those things because nobody outside your team knows your framework. If you need to hire more developers you will need to train them and probably the documentation of your framework will be very poor (or maybe it doesn't exits) so the answer is difficult.

Another important fact is: When we build our own _things_, those _things_ are something like our children and we work with them in a different way than if our boss imposes those _things_ to us as a mandatory things.

**Conclusion**

There are two possible answer: yes and not. Use one or another according to your personal situation. Probably you will choose a wrong answer but remember its better to choose one than stay doing nothing.
