---
layout: post
title: "Taking photos with a phonegap/cordova application and uploading them to the server."
date: "2013-10-28"
tags: 
  - "android"
  - "cordova"
  - "phonegap"
  - "technology"
  - "camera"
  - "filetransfer"
  - "php"
  - "upload"
---

Last days I've working in a phonegap/cordova project. The main purpose of the project was taking photos with the device's camera and sending them to the server. It's a simple apache [cordova](http://cordova.apache.org/) project using the [camera plugin](http://cordova.apache.org/docs/en/3.1.0/cordova_camera_camera.md.html#Camera). According to the documentation we can upload pictures with the following javascript code:

```javascript
navigator.camera.getPicture(onSuccess, onFail, { 
    quality: 100,
    destinationType: Camera.DestinationType.DATA_URL
});
 
function onSuccess(imageData) {
    // here we can upload imageData to the server
}
 
function onFail(message) {
    alert('Failed because: ' + message);
}
```

As we can see we our plugin retrieves a base64 encoded version of our image and we can send it to the server, using jQuery for example

```javascript
function onSuccess(imageData) {
  $.post( "upload.php", {data: imageData}, function(data) {
    alert("Image uploaded!");
  });
}
```

Our server side is trivial. We only need to read the request variable 'data', perform a base64 decode and we have our binary picture ready to be saved.

I test it with an old android smartphone (with a not very good camera) and it's works, but when I tried to use it with a better android phone (with 8mpx camera) it hangs on and it didn't work. After property reading the documentation I realized that it isn't the better way to upload files to the server. It only works if the image file is small. Base64 increases the size of the image and our device can have problems handling the memory. This way Is also slow as hell.

The best way (the way that works, indeed) is, instead of sending the base64 files to the server, to save them into a device's temporary folder and send them using the [file transfer](http://cordova.apache.org/docs/en/3.1.0/cordova_file_file.md.html#FileTransfer) plugin.

```javascript
var pictureSource;   // picture source
var destinationType; // sets the format of returned value
 
document.addEventListener("deviceready", onDeviceReady, false);
 
function onDeviceReady() {
    pictureSource = navigator.camera.PictureSourceType;
    destinationType = navigator.camera.DestinationType;
}
 
function clearCache() {
    navigator.camera.cleanup();
}
 
var retries = 0;
function onCapturePhoto(fileURI) {
    var win = function (r) {
        clearCache();
        retries = 0;
        alert('Done!');
    }
 
    var fail = function (error) {
        if (retries == 0) {
            retries ++
            setTimeout(function() {
                onCapturePhoto(fileURI)
            }, 1000)
        } else {
            retries = 0;
            clearCache();
            alert('Ups. Something wrong happens!');
        }
    }
 
    var options = new FileUploadOptions();
    options.fileKey = "file";
    options.fileName = fileURI.substr(fileURI.lastIndexOf('/') + 1);
    options.mimeType = "image/jpeg";
    options.params = {}; // if we need to send parameters to the server request
    var ft = new FileTransfer();
    ft.upload(fileURI, encodeURI("http://host/upload"), win, fail, options);
}
 
function capturePhoto() {
    navigator.camera.getPicture(onCapturePhoto, onFail, {
        quality: 100,
        destinationType: destinationType.FILE_URI
    });
}
 
function onFail(message) {
    alert('Failed because: ' + message);
}
```

I just realized that sometimes it fails. It looks like a bug of cordova plugin (I cannot assert it), because of that, if you read my code, you can see that if fp.upload fails I retry it (only once). With this little hack it works like a charm (and it's also fast).

The server part is pretty straightforward. We only need to handle uploaded files. Here a minimalistic example with php

```php
<?php
move_uploaded_file($_FILES["file"]["tmp_name"], '/path/to/file');
```

And that's all. We can easily upload our photos from our smartphone using phonegap/cordova.

You can read more articles about cordova [here](http://gonzalo123.com/category/technology/cordova/).
