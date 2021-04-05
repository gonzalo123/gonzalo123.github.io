---
layout: post
title: "Multiple Phonegap Push Notifications in the Android's status bar"
date: "2013-09-30"
categories: 
  - "android"
  - "phonegap"
  - "technology"
tags: 
  - "push-notifications"
---

Last month I worked within an Android project using [Phonegap](http://phonegap.com/), [jQuery Mobile](http://jquerymobile.com/) and [Push Notifications](https://github.com/phonegap-build/PushPlugin). I also wrote one [post](http://gonzalo123.com/2013/08/05/sending-android-push-notifications-from-php-to-phonegap-applications/) explaining how to use PHP to send the server side's part of the push notifications. Today I want to show one small hack, that I've done to change the default behaviour of push notifications. Let me explain it a little bit:

When you use the Push Plugin "out of the box" you will see one message in your Android's status bar everytime we send one push notification (and you application isn't running at this moment). If you click on the notification you application will start and you can handle this notification. But if you send more than one notifications to the device, only the last one will be shown on the status bar. This behaviour can be suitable for most situations, but within my application I wanted to see all the notifications in the status bar until I click on one (then all must disappear). If we want to do that we need to hack a little bit our Phonegap application. Let me show you what I've done.

Basically we need to change **com.plugin.gcm.GCMIntentService** file. If we open this Java file we can see that there's one constant called: _NOTIFICATION\_ID_ and a public function called _createNotification_ with something like that:

```java
public static final int NOTIFICATION_ID = 237;
...
 
public void createNotification(Context context, Bundle extras)
{
    ...
    mNotificationManager.notify((String) appName, NOTIFICATION_ID, mBuilder.build());
 
}
```

I'm not a Java expert, but I notice that if I change this function to:

```java
public static final int NOTIFICATION_ID = 237;
public static int MY_NOTIFICATION_ID = 237;
...
 
public void createNotification(Context context, Bundle extras)
{
    ...
    MY_NOTIFICATION_ID++;
    mNotificationManager.notify((String) appName, MY_NOTIFICATION_ID, mBuilder.build());
 
}
```

Now my android device will show multiple notifications, exactly as I need.

If we need to handle properly the way that our notifications are cancelled we also need to modify the public function _cancelNotification_

```java
public static void cancelNotification(Context context)
{
    NotificationManager mNotificationManager = (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
    //mNotificationManager.cancel((String)getAppName(context), MY_NOTIFICATION_ID);
    mNotificationManager.cancelAll();
    MY_NOTIFICATION_ID = NOTIFICATION_ID;
}
```

And that's all. Multiple notifications as I needed.
