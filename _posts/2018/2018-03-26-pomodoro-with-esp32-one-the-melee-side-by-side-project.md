---
layout: post
title: "Pomodoro with ESP32. One \"The Melee - Side by side\" project"
date: "2018-03-26"
categories: 
  - "js"
  - "node-js"
  - "technology"
  - "websockets"
tags:
  - "javascript"
  - "angular"
  - "arduino"
  - "esp32"
  - "iot"
  - "mqtt"
  - "socket-io"
  - "the-melee"
coverImage: "pomodoro-esp32.png"
---

Last weekend there was a great event called [The Melee - Side by side](http://themelee.org/post/171755270566/the-m%C3%AAl%C3%A9e-side-by-side) (Many thanks to [@ojoven](https://twitter.com/ojoven) and [@diversius](https://diversius.com/)).

The event was one kind of Hackathon where a group of people meet together one day, to share our side projects and to work together (yes. We also have a lunch and beers also :). The format of the event is just a copy of the event that our colleagues from Bilbao called "[El Comité](https://twitter.com/elcomite_bio)".

[@ibaiimaz](https://twitter.com/ibaiimaz) spoke about one project to create one collaborative pomodoro where the people of one team can share their status and see the status of the rest of the team. When I heard pomodoro and status I immediately thought in one servo moving a flag and some LEDs turning on and off. We had a project. [@penniath](https://twitter.com/penniath) and [@tatai](https://twitter.com/tatai) also joined us. We also had a team.

We had a project and we also had a deadline. We must show a working prototype at the end of the day. That means that we didn't have too many time. First we decided the mockup of the project, reducing the initial scope (more ambitious) to fit it within our time slot. We discuss intensely for 10 minutes and finally we describe an ultra detailed blueprint. That's the full blueprint of the project:

![](/assets/images/blueprint.png)

It was time to start working.

@penniath and @tatai worked in the Backend. It must be the responsible of the pomodoro timers, listen to MQTT events and create an API for the frontend. The backend also must provide a WebSockets interface to allow real time events within the frontend. They decided to use node and socket.io for the WebSockets. You can see the source code [here](https://github.com/penniath/pomodoro-api).

@ibaiimaz started with the frontend. He decided to create an Angular web application listening to socket.io events to show the status of the pomodoro. You can see the source code [here](https://github.com/ibaiimaz/Pomoled).

Finaly I worked with the hardware. I created a prototype with one ESP32, two RGB LEDs, one button, one servo and a couple of resistors.

![](/assets/images/fritzing.png)

![](/assets/images/circuit.png)

That's the source code.

```c
#include <WiFi.h>
#include <PubSubClient.h>
 
int redPin_g = 19;
int greenPin_g = 17;
int bluePin_g = 18;
 
int redPin_i = 21;
int greenPin_i = 2;
int bluePin_i = 4;
 
#define SERVO_PIN 16
 
const int buttonPin = 15;
int buttonState = 0;
 
int channel = 1;
int hz = 50;
int depth = 16;
 
const char* ssid = "SSID";
const char* password = "password";
const char* server = "192.168.1.105";
const char* topic = "/pomodoro/+";
const char* clientName = "com.gonzalo123.esp32";
 
WiFiClient wifiClient;
PubSubClient client(wifiClient);
 
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
      client.subscribe(topic);
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      delay(5000);
    }
  }
}
 
void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
 
  String data;
  for (int i = 0; i < length; i++) {
    data += (char)payload[i];
  }
 
  int value = data.toInt();
 
  if (strcmp(topic, "/pomodoro/gonzalo") == 0) {
    Serial.print("[gonzalo]");
    switch (value) {
      case 1:
        ledcWrite(1, 3400);
        setColor_g(0, 255, 0);
        break;
      case 2:
        setColor_g(255, 0, 0);
        break;
      case 3:
        ledcWrite(1, 6400);
        setColor_g(0, 0, 255);
        break;
    }
  } else {
    Serial.print("[ibai]");
    switch (value) {
      case 1:
        setColor_i(0, 255, 0);
        break;
      case 2:
        setColor_i(255, 0, 0);
        break;
      case 3:
        setColor_i(0, 0, 255);  // green
        break;
    }
  }
 
  Serial.print("] value:");
  Serial.println(data);
}
 
void setup()
{
  Serial.begin(115200);
 
  pinMode(buttonPin, INPUT_PULLUP);
  pinMode(redPin_g, OUTPUT);
  pinMode(greenPin_g, OUTPUT);
  pinMode(bluePin_g, OUTPUT);
 
  pinMode(redPin_i, OUTPUT);
  pinMode(greenPin_i, OUTPUT);
  pinMode(bluePin_i, OUTPUT);
 
  ledcSetup(channel, hz, depth);
  ledcAttachPin(SERVO_PIN, channel);
  wifiConnect();
  client.setServer(server, 1883);
  client.setCallback(callback);
 
  delay(1500);
}
 
void mqttEmit(String topic, String value)
{
  client.publish((char*) topic.c_str(), (char*) value.c_str());
}
 
void loop()
{
  if (!client.connected()) {
    mqttReConnect();
  }
 
  client.loop();
 
  buttonState = digitalRead(buttonPin);
  if (buttonState == HIGH) {
    mqttEmit("/start/gonzalo", (String) "3");
  }
 
  delay(200);
}
 
void setColor_i(int red, int green, int blue)
{
  digitalWrite(redPin_i, red);
  digitalWrite(greenPin_i, green);
  digitalWrite(bluePin_i, blue);
}
 
void setColor_g(int red, int green, int blue)
{
  digitalWrite(redPin_g, red);
  digitalWrite(greenPin_g, green);
  digitalWrite(bluePin_g, blue);
}
```

The MQTT server (a [mosquitto](https://mosquitto.org/) server) was initially running in my laptop but as well as I had one Raspberry Pi Zero also in my bag we decided to user the Pi Zero as a server and run mosquitto MQTT server with Raspbian. Everything is better with a Raspberry Pi. @tatai helped me to set up the server.

Here you can see the prototype in action

[![youtube](https://img.youtube.com/vi/yfV9qYOHiSY/0.jpg)](https://www.youtube.com/watch?v=nhncDjXMeQ4)

That's the kind of side projects that I normally create alone but definitely it's more fun to do it with other colleagues even it I need to wake up early one Saturday morning.

Source code of ESP32 [here](https://github.com/gonzalo123/pomodoro.esp32).
