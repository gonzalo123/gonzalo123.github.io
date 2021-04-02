---
layout: post
permalink: /:year/:month/:day/:title/
title: "fetching book cover with google API and dojo"
date: "2009-04-10"
categories: 
  - "dojo"
  - "technology"
  - "web-development"
---

I want to put the cover of one book into my dojo application. I can use amazon Web service but google API has a nice JSONP interface and also google API uses ISBN instead ASIN to fetch book info and for me is easier to know the ISBN (it is in the first page of every book). I have a previous [post](http://gonzalo123.wordpress.com/2009/01/20/jsonp-json-with-padding/) explaining how to use use JSONP with javascript, but now I want to do the same in a dojo-way using dojo.io.script.get

```javascript
dojo.io.script.get({
    url:"http://books.google.com/books?bibkeys=" + isbn + "&jscmd=viewapi",
    callbackParamName: "callback",
    load: dojo.hitch(this, function(booksInfo){googleCallback(booksInfo);})
});
```

And finally we are going to use _booksInfo_ to to show our flaming cover:

```javascript
googleCallback: function(booksInfo) {
        var div = dojo.byId('divId');
        div.innerHTML = '';
        var mainDiv = dojo.doc.createElement('div');
        var x = 0;
        for (i in booksInfo) {
            // Create a DIV for each book
            var book = booksInfo[i];
            var thumbnailDiv = dojo.doc.createElement('div');
            thumbnailDiv.className = "thumbnail";
 
            // Add a link to each book's informtaion page
            var a = dojo.doc.createElement("a");
            a.href = book.info_url;
            a.target = '_blank';
 
            // Display a thumbnail of the book's cover
            var img = dojo.doc.createElement("img");
            img.src = book.thumbnail_url + '&zoom=1';
            img.border = 0;
            a.appendChild(img);
            thumbnailDiv.appendChild(a);
 
            mainDiv.appendChild(thumbnailDiv);
        }
        div.appendChild(mainDiv);
}
```
