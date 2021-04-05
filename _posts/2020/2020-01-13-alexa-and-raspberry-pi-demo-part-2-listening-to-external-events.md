---
layout: post
permalink: /:year/:month/:day/:title/
title: "Alexa and Raspberry Pi demo (Part 2). Listening to external events"
date: "2020-01-13"
categories: 
  - "technology"
tags: 
  - "alexa"
  - "arduino"
  - "esp32"
  - "iot"
  - "javascript"
  - "raspberry-pi"
---

Today I want to keep on with the [previous example](https://gonzalo123.com/2019/12/16/alexa-and-raspberry-pi-demo/). This time I want to create one Alexa skill that listen to external events. The example that I've build is the following one:

I've got one ESP32 with a proximity sensor (one HC-SR04) that is sending the distance captured each 500ms to a MQTT broker (a mosquitto server).

I say "Alexa, use event demo" then Alexa tell me to put my hand close to the sensor and it will tell me the distance.

That's the ESP32 code

```c
#include <WiFi.h>
#include <PubSubClient.h>
 
const char* ssid = "MY_SSID";
const char* password = "MY_PASSWORD";
const char* server = "mqtt.server.ip";
const char* topic = "/alert";
const char* clientName = "com.gonzalo123.esp32";
 
WiFiClient wifiClient;
PubSubClient client(wifiClient);
 
int trigPin = 13;
int echoPin = 12;
int alert = 0;
int alertThreshold = 100;
long duration, distance_cm;
 
void wifiConnect() {
  Serial.print("Connecting to ");
  Serial.println(ssid);
 
  WiFi.begin(ssid, password);
 
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print("*");
  }
 
  Serial.print("WiFi connected: ");
  Serial.println(WiFi.localIP());
}
 
void mqttReConnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    if (client.connect(clientName)) {
      Serial.println("connected");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      delay(5000);
    }
  }
}
 
void mqttEmit(String topic, String value)
{
  client.publish((char*) topic.c_str(), (char*) value.c_str());
}
 
void setup() {
  Serial.begin(9600);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
 
  wifiConnect();
  client.setServer(server, 1883);
 
  delay(1500);
}
 
void loop() {
  if (!client.connected()) {
    mqttReConnect();
  }
 
  client.loop();
 
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
 
  duration = pulseIn(echoPin, HIGH) / 2;
  distance_cm = duration / 29;
 
  if (distance_cm <= alertThreshold && alert == 0) {
      alert = 1;
      Serial.println("Alert!");
      mqttEmit("/alert", (String) distance_cm);
  } else if(distance_cm > alertThreshold && alert == 1) {
    alert = 0;
    Serial.println("No alert");
  }
 
  Serial.print("Distance: ");
  Serial.print(distance_cm);
  Serial.println(" cm ");
 
  delay(500);
}
```

And this is my alexa skill:

```javascript
'use strict'
 
const Alexa = require('ask-sdk-core')
 
const RequestInterceptor = require('./interceptors/RequestInterceptor')
const ResponseInterceptor = require('./interceptors/ResponseInterceptor')
const LocalizationInterceptor = require('./interceptors/LocalizationInterceptor')
const GadgetInterceptor = require('./interceptors/GadgetInterceptor')
 
const YesIntentHandler = require('./handlers/YesIntentHandler')
const NoIntentHandler = require('./handlers/NoIntentHandler')
const LaunchRequestHandler = require('./handlers/LaunchRequestHandler')
const CancelAndStopIntentHandler = require('./handlers/CancelAndStopIntentHandler')
const SessionEndedRequestHandler = require('./handlers/SessionEndedRequestHandler')
const CustomInterfaceEventHandler = require('./handlers/CustomInterfaceEventHandler')
const CustomInterfaceExpirationHandler = require('./handlers/CustomInterfaceExpirationHandler')
 
const FallbackHandler = require('./handlers/FallbackHandler')
const ErrorHandler = require('./handlers/ErrorHandler')
 
let skill
exports.handler = function (event, context) {
  if (!skill) {
    skill = Alexa.SkillBuilders.custom().
      addRequestHandlers(
        LaunchRequestHandler,
        YesIntentHandler,
        NoIntentHandler,
        CustomInterfaceEventHandler,
        CustomInterfaceExpirationHandler,
        CancelAndStopIntentHandler,
        SessionEndedRequestHandler,
        FallbackHandler).
      addRequestInterceptors(
        RequestInterceptor,
        ResponseInterceptor,
        LocalizationInterceptor,
        GadgetInterceptor).
      addErrorHandlers(ErrorHandler).create()
  }
  return skill.invoke(event, context)
}
```

The process is similar to the previous example. There's a GadgetInterceptor to find the endpointId of my Raspberry Pi. But now the Raspberry Pi must emit to the event to the skill, not the skill to the event. Now my rpi's python script is a little bit more complicated. We need to start the main loop for the Alexa Gadget SDK but also we need to start another loop listening to a mqtt event. We need to use threads as we see in a previous [post](https://gonzalo123.com/2019/10/07/playing-with-threads-and-python-part-2/). That's the script that I'm using.

```python
from queue import Queue, Empty
import threading
import logging
import sys
import paho.mqtt.client as mqtt
from agt import AlexaGadget
 
logging.basicConfig(stream=sys.stdout, level=logging.DEBUG)
logger = logging.getLogger(__name__)
 
 
class Listener(threading.Thread):
    def __init__(self, queue=Queue()):
        super(Listener, self).__init__()
        self.queue = queue
        self.daemon = True
 
    def on_connect(self, client, userdata, flags, rc):
        print("Connected!")
        client.subscribe("/alert")
 
    def on_message(self, client, userdata, msg):
        self.on_event(int(msg.payload))
 
    def on_event(self, cm):
        pass
 
    def run(self):
        client = mqtt.Client()
        client.on_connect = self.on_connect
        client.on_message = self.on_message
 
        client.connect('192.168.1.87', 1883, 60)
        client.loop_forever()
 
 
class Gadget(AlexaGadget):
    def __init__(self):
        super().__init__()
        Listener.on_event = self._emit_event
 
    def _emit_event(self, cm):
        logger.info("_emit_event {}".format(cm))
        payload = {'cm': cm}
        self.send_custom_event('Custom.gonzalo123', 'sensor', payload)
 
 
l = Listener()
l.start()
 
 
def main():
    gadget = Gadget()
    gadget.main()
 
 
if __name__ == '__main__':
    main()
```

[![youtube](https://img.youtube.com/vi/5DPtZTILHR8/0.jpg)](https://www.youtube.com/watch?v=5DPtZTILHR8)

Source code in my [github](https://github.com/gonzalo123/alexa_rpi_event)
