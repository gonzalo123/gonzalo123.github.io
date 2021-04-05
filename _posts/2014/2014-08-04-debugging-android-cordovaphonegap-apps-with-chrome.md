---
layout: post
title: "Debugging android cordova/phonegap apps with Chrome"
date: "2014-08-04"
tags: 
  - "angularjs"
  - "cordova"
  - "phonegap"
  - "technology"
  - "chrome"
  - "js"
  - "tip"
---

Maybe this post can be obvious but I've spoken about it with various developers who don't know it. It really improves the developing process of cordova/phonegap apps with android at least for me.

With android we can see the log with "adb logcat" but it's a nightmare. Huge amount of information about our app and also about the operating system. If we're grep ninjas we can handle it, but as well as I'm not a ninja I prefer another solution. Do you know "chrome://inspect/"? If not, have a look as soon as possible to this tool. We can see the browser's console of our android in our desktop browser. We only need to enable "usb remote debugger" within our android device and plug with a USB cable. Chrome will detect the remote browser and we can see the console in the same way than we see it when we use Chrome locally.

But we're speaking about cordova/phonegap apps here so, what we need to do to use chrome://inspect with our hybrid apps? The answer is simple: we don't need to do anything. Cordova applications is nothing than a Webkit browser inside a native app. Chrome es Webkit too so chrome://inspect will detect our remote device app and we will open console.


![Inspect_with_Chrome_Developer_Tools_and_bad_religion-the_gray_race](/assets/images/inspect_with_chrome_developer_tools_and_bad_religion-the_gray_race1.jpg)

This small trick in addition to the last [post](http://gonzalo123.com/2014/07/21/testing-phonegapcordova-applications-fast-as-hell-in-the-device-with-ionic-framework/ "Testing Phonegap/Cordova applications fast as hell in the device (with ionic framework)") really marks a before and an after at least in my developing process.

If our app crashes in the device we only need to see the console's log within our browser and see what happens. We also can add functionality, change variables, and override functions in the same way than we do it with our local browser.
