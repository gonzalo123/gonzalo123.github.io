---
layout: post
title: "Encrypt Websocket (socket.io) communications"
date: "2016-09-26"
tags: 
  - "node-js"
  - "npm"
  - "socket-io"
  - "technology"
  - "crypt"
  - "js"
  - "nodejs"
  - "websockets"
---

I'm a big fan of WebSockets and socket.io. I've written a lot of [about](https://gonzalo123.com/category/technology/socket-io/) [it](https://gonzalo123.com/category/technology/websockets/). In last posts I've [written](https://gonzalo123.com/2016/06/06/sharing-authentication-between-socket-io-and-a-php-frontend-using-json-web-tokens/) about socket.io and authentication. Today we're going to speak about communications.

Imagine we've got a websocket server and we connect our application to this server (even using https/wss). If we open our browser's console we can inspect our WebSocket communications. We also can enable [debugging](http://socket.io/docs/logging-and-debugging/). This works in a similar way than when we start the [promiscuous](https://en.wikipedia.org/wiki/Promiscuous_mode) mode within our network interface. We will see every packets. Not only the packets that server is sending to us.

If we send send sensitive information over websockets, that means than one logged user can see another ones information. We can separate [namespaces](http://socket.io/docs/rooms-and-namespaces/) in our socket.io server. We also can do another thing: Encrypt communications using [crypto-js](https://www.npmjs.com/package/crypto-js).

I've created one small wrapper to use it with socket.io. We can install our server dependency

```commandline
npm g-crypt
```

And install our client dependency with bower

```commandline
bower install g-crypt
```

And use it in our server

```javascript
var io = require('socket.io')(3000),
    Crypt = require("g-crypt"),
    passphrase = 'super-secret-passphrase',
    crypter = Crypt(passphrase);
 
io.on('connection', function (socket) {
    socket.on('counter', function (data) {
        var decriptedData = crypter.decrypt(data);
        setTimeout(function () {
            console.log("counter status: " + decriptedData.id);
            decriptedData.id++;
            socket.emit('counter', crypter.encrypt(decriptedData));
        }, 1000);
    });
});
```

And now a simple HTTP application

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
Open console to see the messages
 
<script src="http://localhost:3000/socket.io/socket.io.js"></script>
<script src="assets/cryptojslib/rollups/aes.js"></script>
<script src="assets/g-crypt/src/Crypt.js"></script>
<script>
    var socket = io('http://localhost:3000/'),
        passphrase = 'super-secret-passphrase',
        crypter = Crypt(passphrase),
        id = 0;
 
    socket.on('connect', function () {
        console.log("connected! Let's start the counter with: " + id);
        socket.emit('counter', crypter.encrypt({id: id}));
    });
 
    socket.on('counter', function (data) {
        var decriptedData = crypter.decrypt(data);
        console.log("counter status: " + decriptedData.id);
        socket.emit('counter', crypter.encrypt({id: decriptedData.id}));
    });
</script>
 
</body>
</html>
```

Now our communications are encrypted and logged user cannot read another ones data.

Library is a simple wrapper

```javascript
Crypt = function (passphrase) {
    "use strict";
    var pass = passphrase;
    var CryptoJSAesJson = {
        parse: function (jsonStr) {
            var j = JSON.parse(jsonStr);
            var cipherParams = CryptoJS.lib.CipherParams.create({ciphertext: CryptoJS.enc.Base64.parse(j.ct)});
            if (j.iv) cipherParams.iv = CryptoJS.enc.Hex.parse(j.iv);
            if (j.s) cipherParams.salt = CryptoJS.enc.Hex.parse(j.s);
            return cipherParams;
        },
        stringify: function (cipherParams) {
            var j = {ct: cipherParams.ciphertext.toString(CryptoJS.enc.Base64)};
            if (cipherParams.iv) j.iv = cipherParams.iv.toString();
            if (cipherParams.salt) j.s = cipherParams.salt.toString();
            return JSON.stringify(j);
        }
    };
 
    return {
        decrypt: function (data) {
            return JSON.parse(CryptoJS.AES.decrypt(data, pass, {format: CryptoJSAesJson}).toString(CryptoJS.enc.Utf8));
        },
        encrypt: function (data) {
            return CryptoJS.AES.encrypt(JSON.stringify(data), pass, {format: CryptoJSAesJson}).toString();
        }
    };
};
 
if (typeof module !== 'undefined' && typeof module.exports !== 'undefined') {
    CryptoJS = require("crypto-js");
    module.exports = Crypt;
} else {
    window.Crypt = Crypt;
}
```

Library available in my [github](https://github.com/gonzalo123/crypt) and also we can use it using npm and bower.
