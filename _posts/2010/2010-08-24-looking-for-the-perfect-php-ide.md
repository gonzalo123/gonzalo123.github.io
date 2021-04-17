---
layout: post
title: "Looking for the perfect PHP IDE"
date: "2010-08-24"
tags: 
  - "php"
categories: 
  - "php"
---

I've got a problem. I haven’t found the perfect IDE for me. Yet. I've got problems with every software. Now I will try to explain the problems I have and maybe someone shows me the light and helps me to discover the perfect editor/IDE.

First important thing. I’m a Linux user so Windows only softwares are out. I know Mac user really love Textmate but buying a mac is not in my scope. They are good hardware, reliable and of course cool. I know nowadays there are a lot of web developers working with mac but I’m not convinced yet to change my computer from PC with linux to mac. Probably I will make myself the question when the core breaks down. Don't ask me why but my last three PCs broke down with a fail the core (after years of usage). So the baseline is clear. Software must work with linux and it must work in a native way. Not with wine or things like that.

My requirement list is the following ones:

- Code auto-completion.
- Syntax highlight.
- Debugger.

Easy, isn’t it? But even with those simple requirements I haven’t found yet my perfect IDE. So I’m going to enumerate the problems I’ve got with each one.

## Zend Studio 7

It’s not free. That’s not the main problem. But we can find another similar for free. Debugger works fine. Code auto-completion really good. Syntax highlight perfect. Auto indentation and one of my favourite feature code analysis. Really useful. It detects problems such as:

```php
$var = 0;
if ($Var == 0) { // $Var and $var are different variables. It’s probably a type error very difficult to detect in a glance
    ...
}
```

It’s look like the perfect IDE but it’s maddeningly slow. If you are working with a few files is perfect but is your project is big (hundred of files) sometimes you are forced to make too many coffee breaks when it becomes crazy (building workspace...). Maybe if you’ve got a super pc with infinite RAM and hundred cores it works fine but, at least for me it’s irritating.

## Eclipse PDT

Zend Studio 7 is based on eclipse. PDT it’s almost the same than Zend Studio 7 (without its great integration with the zend services that I don’t use). Exactly the same problems (very slow) but even without some extra features, such as code analysis (really useful for me). By other hand it’s free (as free beer)

## Netbeans

Similar features than Eclipse PDT and lighter. Also free. The problem here it’s the debugger. Only works with xdebug. Not a real problem for me. I don’t really mind what debugger to use. Zend debugger and xdebug meets my requirements. But it’s a bit strange how to debug. I don’t like it. In early versions it created the url when you started to debug. And that url cannot be changed properly. That’s means it doesn’t worked for me. Now it’s better but still not good. It looks like the PHP plugin of Netbeans isn’t as good as Java one’s. The future of Netbeans isn’t clear with the acquisition of Oracle. They said Netbeans isn’t an strategical project (that’s means they cut some funds)

## VIM

It looks like the perfect editor but It’s hard to learn, at least in the beginning. Syntax highlight perfect. Really light. Works perfect everywhere even on remote hosts via ssh. Code auto-completion works but not as good as in eclipse and Netbeans (but it’s fair enough). I said is not as good as Eclipse and Netbeans because for example it doesn’t hint me with the variables of the function or ignores PHPDoc in the auto-completion pop up, and I really appreciate it. The main problem I have is with the debugger. It’s so strange for me. I thing I need a couple of hours working on it because nowadays debugging with VIM is something like a miracle a miracle for me. Sometimes work but I it’s too many endeavor. Yet. I hope so. If you want more about about PHP and VIM, you must take a look to this [link](http://zmievski.org/2010/06/vim-for-programmers-on-slideshare).

## Emacs

I must admit that I am too lazy to learn to use it. I invested time with VIM. I feel that I need more and I don’t want to start with another one like emacs from zero. Maybe I’m wrong and that’s the perfect IDE but it’s not in my mind now.

## Zend Studio 5.5

That “was” the perfect IDE. It was not for free but it was really good. Fast. Even with big projects (sometimes become crazy and crashed, I known). Debugging was perfect. Only with Zend debugger but really easy to use. You don’t need a great pc. 2G RAM was fair enough. But there’s a problem. This software is obsolete. Zend changed to an Eclipse based one. We can still using it but there isn’t any update. The main problem with it is we cannot use any new PHP5.3 features, such as namespaces. OK we can use them if our server has PHP5.3 but IDE mark those new keywords as syntax errors. So If we work with this software and a PHP5.3 project we need to assume that the red warnings of our editor (syntax errors) are not always errors. It’d be a great new for me if Zend people releases Zend Studio 5.5 as open source a someone continues the project adding PHP 5.3 support.

As you can see I’m not 100% happy with any IDE. Do you have a perfect IDE? I’d really like to be wrong in my assumptions. So I’ll keep looking for.
