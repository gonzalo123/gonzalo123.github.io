---
layout: post
title: "NFC tag reader with Arduino"
date: "2017-06-12"
categories: 
  - "arduino"
  - "node-js"
  - "technology"
tags: 
  - "iot"
  - "node"
---

Today I want to use the NFC tag reader module with my Arduino. The idea is build a simple prototype to read NFC tags and validate them against a remote server (for example a node tcp server). Depending on the tag we'll trigger one digital output or another. In the example we'll connect leds to those outputs, but in the real life we can open door or something similar.

We'll use a MFRC522 module. It's a cheap Mifare RFID/NFC tag reader and writer.

MFRC522 Connection:

- sda: 10 (\*) -> 8
- sck: 13
- Mosi: 11
- Miso: 12
- Rq: --
- Gnd: Gnd
- Rst: 9
- 3.3V: 3.3V

In this example we'll use a ethernet shield to connect our Arduino board to the LAN. We must take care with it. If we use ethernet shield with a MFRC522 there's a SPI conflict (due to ethernet shield's SD card reader). We need to use another SDA pin (here I'm using pin 8 instead of 10) and disable w5100 SPI before configure ethernet.

```c
// disable w5100 SPI
pinMode(10, OUTPUT);
digitalWrite(10, HIGH);
```

Here is the Arduino code

```c
#include <SPI.h>
#include <MFRC522.h>
#include <Ethernet.h>
#include <EthernetClient.h>
 
#define RST_PIN 9
#define SS_PIN  8
#define ERROR_PIN 7
#define OPEN_PIN 6
#define OPEN_DELAY 2000
 
char server[] = "192.168.1.104";
int port = 28001;
 
signed long timeout;
 
byte mac[] = { 0x00, 0xAA, 0xBB, 0xCC, 0xDE, 0x02 };
MFRC522 mfrc522(SS_PIN, RST_PIN);
EthernetClient client;
 
void printArray(byte *buffer, byte bufferSize) {
  for (byte i = 0; i < bufferSize; i++) {
    Serial.print(buffer[i] < 0x10 ? " 0" : " ");
    Serial.print(buffer[i], HEX);
  }
}
 
String dump_byte_array(byte *buffer, byte bufferSize) {
          String out = "";
    for (byte i = 0; i < bufferSize; i++) {
        out += String(buffer[i] < 0x10 ? " 0" : " ") + String(buffer[i], HEX);
    }
    out.toUpperCase();
    out.replace(" ", "");
     
    return out;
}
 
void resetLeds() {
  digitalWrite(OPEN_PIN, LOW);
  digitalWrite(ERROR_PIN, LOW);
}
 
void open() {
  Serial.println("OPEN!");
  digitalWrite(OPEN_PIN, HIGH);
  delay(OPEN_DELAY);
  digitalWrite(OPEN_PIN, LOW);
}
 
void error() {
  Serial.println("ERROR!");
  digitalWrite(ERROR_PIN, HIGH);
  delay(OPEN_DELAY);
  digitalWrite(ERROR_PIN, LOW);
}
 
void scanCard() {
  byte status;
  byte buffer[18];
  int err = 0;
  byte size = sizeof(buffer);
  EthernetClient c;
       
  if (mfrc522.PICC_IsNewCardPresent()) {
    if (mfrc522.PICC_ReadCardSerial()) {
      const String ID = dump_byte_array(mfrc522.uid.uidByte, mfrc522.uid.size);
      Serial.println("New tag read: " + ID);
      mfrc522.PICC_HaltA();
      
      if (client.connect(server, port)) {
        timeout = millis() + 3000;
        client.println("OPEN:" + ID);
        delay(10);
 
        while(client.available() == 0) {
          if (timeout - millis() < 1000) {
              error();
              goto close;
          }
        } 
        int size;
        bool status;
         
        while((size = client.available()) > 0) {
          uint8_t* msg = (uint8_t*)malloc(size);
          size = client.read(msg,size);
          //Serial.write(msg, size);
          // 4F4B   -> OK
          // 4E4F4B -> NOK
          status = dump_byte_array(msg, size) == "4F4B";
          free(msg);
        }
         
        Serial.println(status ? "OK!" : "NOK!");
        if (status) {
          open();
        } else {
          error();
        }
close:
        client.stop();
      } else {
        Serial.println("Connection Error");
        error();
      }
    }
  }
}
 
void setup()
{
  resetLeds();
  Serial.begin(9600);
  Serial.println("Setup ...");
 
  // disable w5100 SPI
  pinMode(10, OUTPUT);
  digitalWrite(10, HIGH);
 
  SPI.begin();
  mfrc522.PCD_Init();
 
  if (Ethernet.begin(mac) == 0) {
    Serial.println("DHCP Error");
    error();
    while (true) {}
  }
  Serial.print("My IP: ");
  for (byte B = 0; B < 4; B++) {
    Serial.print(Ethernet.localIP()[B], DEC);
    Serial.print(".");
  }
  Serial.println();
  Serial.println("Finish setup");
  timeout = 0;
}
 
void loop()
{
  resetLeds();
  scanCard();
  delay(200);
} 
```

```javascript
var net = require('net');
 
var LOCAL_PORT = 28001;
var validTags = ['X3C86AD9'];
 
var validateTag = function(tag) {
    return validTags.indexOf(tag) > -1;
};
 
var server = net.createServer(function (socket) {
    console.log(socket.remoteAddress + ":" + socket.remotePort);
    socket.on('data', function(msg) {
        var out;
        [action, tag] = msg.toString().toUpperCase().trim().split(":");
        console.log(action, tag);
        switch (action) {
            case 'OPEN':
                out = validateTag(tag) ? "OK" : "NOK";
                console.log(out);
                socket.write(out);
                break;
            default:
                console.log("unknown action:", action);
        }
 
        socket.destroy();
    });
});
 
server.listen(LOCAL_PORT, '0.0.0.0');
```

And that's all. Here a video with the working example

[![youtube](https://img.youtube.com/vi/hV4BeSx0Kw4/0.jpg)](https://www.youtube.com/watch?v=hV4BeSx0Kw4)

And full code available in my [github](https://github.com/gonzalo123/arduino-nfc-reader) account.

References about rfid and Arduino: [here](https://www.luisllamas.es/arduino-rfid-mifare-rc522/), [here](http://hetpro-store.com/TUTORIALES/modulo-lector-rfid-rc522-rf-con-arduino/) and [here](https://forum.arduino.cc/index.php?topic=198768.0)
