---
layout: post
title: "My VIM configuration for PHP development."
date: "2011-01-03"
tags: 
  - "php"
  - "technology"
  - "tips"
  - "vim"
---

Keeping on with my continuous search for the perfect IDE I’ve resumed my fight against VIM. As someone told me the learning of vim is a road of pain. It’s something like going to the gym. We known that going to the gym is good and healthy but it’s hard and painfully especially at the beginning. The learning curve of vim is hard. Vim is rare, or at least it’s different than all other IDEs. In this post I don’t want to create a vim tutorial (there are a lot of on the web). I want to share my actual vim configuration and the plugins I normally use. The configuration is not very original. It’s based on Matthew Weier O'Phinney configuration. You can find here a great [post](http://weierophinney.net/matthew/archives/249-Vim-Toolbox,-2010-Edition.html) about vim in Matthew Weier’s blog. VIM is not my default IDE yet. I hope to swap to VIM soon but I feel myself slower when I code comparing with Netbeans or ZendStudio, and slow means less productive but I also feel if invest more time vim can be really productive. Here we are my vim plug-ins:

## **NerdTree**

Great [plug-in](http://www.vim.org/scripts/script.php?script_id=1658). Allows to explore filesystem. It’s easy and fast. I use _F5_ to toogle the NerdTree We move across the filesystem (_jk_) and open files with “_o_”. It’s really intuitive. Another great feature of NerdTree are the bookmarks. We select a folder and we type :Bookmark and we easily create a bookmark. Really cool when we work with different projects.

[](/assets/images/vim.png "vim")

## **BufferExplorer**

We can manage vim’s buffers with standard vim commands(:ls) but I like to use this extension. I’ve mapped BufferExplorer with ç key. I use a spanish keyboard and _ç_ key is a letter I never use and is easily accessible, even for my small hands. When I type _ç_ I can switch between my buffers.

## **Zen Coding**

Really useful when writing HTML pre#myid>span <Conrol-y> and we get:

```html
<pre id="myid"></pre>
```

You can see more examples [here](http://code.google.com/p/zen-coding/wiki/ZenHTMLSelectorsEn)

## **Debugger**

We need xdebug plugin installed on our server. I’ve changed the original keys for this plugin because it uses NerdTree ones. I use control + the original keys. For example control + F5 starts the debuger and waits for the debug session from the browser. Xdebug helper for google chrome works fine.

[](/assets/images/vimdebuger.png "vimdebuger")

## **snipMate**

Mac developers love Textmate. I’ve never use textmate (I’m a linux user) but [this](http://www.vim.org/scripts/script.php?script_id=2540) allow us create snippets features in Vim like Textmate’s snippets. It’s a good practice to create the snippets we normally use to increase our productivity. I’ve created several snippets for creating classes, function, for loops, and common pieces of code like that

## **Command-t**

We can open files with :e but commnad-t plugin helps us to open files. I trigger command-t with . Leader is an especial key in vim. We can map this key to a letter. I’ve mapped to “,”

## **Omni completion**

control x - control o. If you have previously created your exuberant tags files, helps you with the autocompletion. Ok it’s not as good a netbeans or eclipse but it works. To create ctags file there is a mkTags bash file under .vim/bin folder

[](/assets/images/vim1.png "vim")

## **Supertab**

snipMate is trigger with tab key but tab key is used to for auto-completion with php plugin (ftp plugin). Supertab plugin magically triggers the plugin we need. I don’t really know how it works but it works (at least as I expect)

## **Another useful commands**

_çç_: (insert mode) I use çç as esc key because escape key is too far for my small fingers _ç_: (normal mode) Buffer explorer _control p_: PHPDoc helper _zz_: Open all forlds _F7_: open close a fold _F6_: show/hide line numbers

You can download my vim configuration from google code [here](http://code.google.com/p/gonzalo123-vim/)

How to configure vim with my setings:

```shell
hg clone https://gonzalo123-vim.googlecode.com/hg/ gonzalo123-vim
ln -s gonzalo123-vim/ ~/.vim
ln -s gonzalo123-vim/vimrc ~/.vimrc
ln -s gonzalo123-vim/gvimrc ~/.gvimrc
```
