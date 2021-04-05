---
layout: post
title: "Sending Android Push Notifications from PHP to phonegap applications"
date: "2013-08-05"
tags: 
  - "android"
  - "phonegap"
  - "php"
  - "technology"
  - "push"
---

Last days I’ve been working within a [Phonegap](http://phonegap.com/) project for Android devices using Push Notifications. The idea is simple. We need to use the Push Notification [Plugin](https://github.com/phonegap-build/PushPlugin) for Android. First we need to register the Google Cloud Messaging for Android service at Google's [console](https://code.google.com/apis/console/), and then we can send Push notifications to our Android device.

The Push Notification plugin provides a simple example to send notifications using [Ruby](https://github.com/phonegap-build/PushPlugin/blob/master/Example/server/pushGCM.rb). Normally my backend is built with PHP (and sometimes Python) so instead of using the ruby script we are going to build a simple PHP script to send Push Notifications.

The script is very simple 
```php
<?php
$apiKey = "myApiKey";
$regId = "device reg ID";
 
$pusher = new AndroidPusher\Pusher($apiKey);
$pusher->notify($regId, "Hola");
 
print_r($pusher->getOutputAsArray());
```

And the whole library you can see here: 

```php
<?php
namespace AndroidPusher;
 
class Pusher
{
    const GOOGLE_GCM_URL = 'https://android.googleapis.com/gcm/send';
 
    private $apiKey;
    private $proxy;
    private $output;
 
    public function __construct($apiKey, $proxy = null)
    {
        $this->apiKey = $apiKey;
        $this->proxy  = $proxy;
    }
 
    /**
     * @param string|array $regIds
     * @param string $data
     * @throws \Exception
     */
    public function notify($regIds, $data)
    {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, self::GOOGLE_GCM_URL);
        if (!is_null($this->proxy)) {
            curl_setopt($ch, CURLOPT_PROXY, $this->proxy);
        }
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $this->getHeaders());
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $this->getPostFields($regIds, $data));
 
        $result = curl_exec($ch);
        if ($result === false) {
            throw new \Exception(curl_error($ch));
        }
 
        curl_close($ch);
 
        $this->output = $result;
    }
 
    /**
     * @return array
     */
    public function getOutputAsArray()
    {
        return json_decode($this->output, true);
    }
 
    /**
     * @return object
     */
    public function getOutputAsObject()
    {
        return json_decode($this->output);
    }
 
    private function getHeaders()
    {
        return [
            'Authorization: key=' . $this->apiKey,
            'Content-Type: application/json'
        ];
    }
 
    private function getPostFields($regIds, $data)
    {
        $fields = [
            'registration_ids' => is_string($regIds) ? [$regIds] : $regIds,
            'data'             => is_string($data) ? ['message' => $data] : $data,
        ];
 
        return json_encode($fields, JSON_HEX_TAG | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_HEX_AMP | JSON_UNESCAPED_UNICODE);
    }
}
```

Maybe we could improve the library with a parser of google's ouptuput, basically because we need to handle this output to notice if the user has uninstalled the app (and we need the remove his reg-id from our database), but at least now it cover all my needs. You can see the code at [github](https://github.com/gonzalo123/androidpusher)
