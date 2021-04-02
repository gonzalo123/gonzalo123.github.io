---
layout: post
permalink: /:year/:month/:day/:title/
title: "Building Bluetooth iot devices compatible with Alexa"
date: "2020-03-30"
categories: 
  - "technology"
tags: 
  - "alexa"
  - "arduino"
  - "bluetooth"
  - "iot"
---

Alexa can speak with iot devices (bulbs, switches, ...) directly without creating any skill. Recently I've discoverer the library [fauxmoesp](https://bitbucket.org/xoseperez/fauxmoesp/src/master/) to use a ESP32 as a virtual device and use it with Alexa.

I want to create a simple example with my M5Stack (it's basically one ESP32 with an screen). It's pretty straightforward to do it. Here the firmware that I deploy to my device

```c
#include <M5Stack.h>
#include <WiFi.h>
#include "fauxmoESP.h"
 
#define SERIAL_BAUDRATE 115200
 
#define WIFI_SSID "my_SSOD"
#define WIFI_PASS "my_password"
 
#define DEVICE_1 "my device"
 
fauxmoESP fauxmo;
bool current_state;
bool change_flag;
int current_value;
 
void wifiSetup() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
 
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(100);
  }
  Serial.println();
  Serial.printf("WIFI coneccted! IP: %s\n", WiFi.localIP().toString().c_str());
}
 
void setup() {
  Serial.begin(SERIAL_BAUDRATE);
  Serial.println();
  M5.begin();
  M5.Power.begin();
 
  wifiSetup();
 
  current_state = false;
  current_value = 0;
  change_flag = false;
   
  fauxmo.createServer(true);
  fauxmo.setPort(80); 
  fauxmo.enable(true);
  fauxmo.addDevice(DEVICE_1);
 
  fauxmo.onSetState([](unsigned char device_id, const char * device_name, bool state, unsigned char value) { 
    Serial.printf("Device #%d (%s) state: %s value: %d\n", device_id, device_name, state ? "ON" : "OFF", value);
    current_value = value * 100 / 254;
    current_state = state;
    change_flag = true;
  });
}
 
void loop() {
  fauxmo.handle();
 
  static unsigned long last = millis();
  if (millis() - last > 5000) {
    last = millis();
    Serial.printf("[MAIN] Free heap: %d bytes\n", ESP.getFreeHeap());
  }
  if (change_flag) {
    M5.Lcd.fillScreen(current_state ? GREEN : RED);
    M5.Lcd.setCursor(100, 100);
    M5.Lcd.setTextColor(WHITE);
    M5.Lcd.setTextSize(6);
    M5.Lcd.print(current_state ? current_value : 0);
    change_flag = false;
  }
}
```

Then I only need to pair my virtual device (in this case "my device") to my compatible Alexa Device (In my case one echo spot). We can do it with the Alexa device's menu or simply saying "Alexa discover devices". Then I've got my iot device paired to my Alexa and I can say something like: "Alexa, switch on my device", "Alexa, switch off my device" or "Alexa, set my device at 50%"

Here a small video showing the working example: 

[![youtube](https://img.youtube.com/vi/6XXFPe\_Fxlw/0.jpg)](https://www.youtube.com/watch?v=6XXFPe\_Fxlw)
