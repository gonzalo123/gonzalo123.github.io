---
layout: post
title: "Playing with arduino, IoT, crossbar and websockets"
date: "2016-12-12"
tags: 
  - "arduino"
  - "html5"
  - "js"
  - "node-js"
  - "python"
  - "technology"
  - "iot"
  - "websockets"
---

Yes. Finally I've got an arduino board. It's time to hack a little bit. Today I want to try different things. I want to display in a webpage one value from my arduino board. For example one analog data using a potentiometer. Let's start.

We are going to use one potentiometer. A potentiometer is a resistor with a rotating contact that forms an adjustable voltage divider. It has three pins. If we connect one pin to 5V power source of our arduino, another one to the ground and another to one A0 (analog input 0), we can read different values depending on the position of potentiometer's rotating contact.

![arduino_analog](/assets/images/arduino_analog.png)

Arduino has 10 bit analog resolution. That means 1024 possible values, from 0 to 1023. So when our potentiometer gives us 5 volts we'll obtain 1024 and when our it gives us 0V we'll read 0. Here we can see a simple arduino program to read this analog input and send data via serial port: 

```c
int mem;
 
void setup() {
  Serial.begin(9600);
}
 
void loop() {
  int value = analogRead(A0);
  if (value != mem) {
    Serial.println(value);
  }
  mem = value;
 
  delay(100);
}
```

This program is simple loop with a delay of 100 milliseconds that reads A0 and if value is different than previously read (to avoid sending the same value when nobody is touching the potentiometer) we send the value via serial port (with 9600 bauds)

We can test our program using the serial monitor of our arduino IDE our using another serial monitor.

Now we're going to create one script to read this serial port data. We're going to use Python. I'll use my laptop and my serial port is /dev/tty.usbmodem14231

```python
import serial
 
arduino = serial.Serial('/dev/tty.usbmodem14231', 9600)
 
while 1:
  print arduino.readline().strip()
```

Basically we've got our backend running. Now we can create a simple frontend.

```html
...
<div id='display'></div>
...
```

We'll need websockets. I normally use socket.io but today I'll use [Crossbar.io](http://crossbar.io/). Since I hear about it in a Ronny's [talk](https://speakerdeck.com/ronnylt/real-time-services-with-silex-and-symfony-desymfony-2016) at deSymfony conference I wanted to use it.

I'll change a little bit our backend to emit one event

```python
import serial
from os import environ
from twisted.internet.defer import inlineCallbacks
from twisted.internet.task import LoopingCall
from autobahn.twisted.wamp import ApplicationSession, ApplicationRunner
 
arduino = serial.Serial('/dev/tty.usbmodem14231', 9600)
 
class SeriaReader(ApplicationSession):
    @inlineCallbacks
    def onJoin(self, details):
        def publish():
            return self.publish(u'iot.serial.reader', arduino.readline().strip())
 
        yield LoopingCall(publish).start(0.1)
 
if __name__ == '__main__':
    runner = ApplicationRunner(environ.get("GONZALO_ROUTER", u"ws://127.0.0.1:8080/ws"), u"iot")
    runner.run(SeriaReader)
```

Now I only need to create a crossbar.io server. I will use node to do it 

```javascript
var autobahn = require('autobahn'),
    connection = new autobahn.Connection({
            url: 'ws://0.0.0.0:8080/ws',
            realm: 'iot'
        }
    );
 
connection.open();
```

And now we only need to connect our frontend to the websocket server

```javascript
$(function () {
    var connection = new autobahn.Connection({
        url: "ws://192.168.1.104:8080/ws",
        realm: "iot"
    });
 
    connection.onopen = function (session) {
        session.subscribe('iot.serial.reader', function (args) {
            $('#display').html(args[0]);
        });
    };
 
    connection.open();
});
```

It works but thre's a problem. The first time we connect with our browser we won't see the display value until we change the position of the potentiometer. That's because 'iot.serial.reader' event is only emitted when potentiometer changes. No change means no new value. To solve this problem we only need to change a little bit our crossbar.io server. We'll "memorize" the last value and we'll expose one method 'iot.serial.get' to ask about this value

```javascript
var autobahn = require('autobahn'),
    connection = new autobahn.Connection({
            url: 'ws://0.0.0.0:8080/ws',
            realm: 'iot'
        }
    ),
    mem;
 
connection.onopen = function (session) {
    session.register('iot.serial.get', function () {
        return mem;
    });
 
    session.subscribe('iot.serial.reader', function (args) {
        mem = args[0];
    });
};
 
connection.open();
```

An now in the frontend we ask for 'iot.serial.get' when we connect to the socket

```javascript
$(function () {
    var connection = new autobahn.Connection({
        url: "ws://192.168.1.104:8080/ws",
        realm: "iot"
    });
 
    connection.onopen = function (session) {
        session.subscribe('iot.serial.reader', function (args) {
            $('#display').html(args[0]);
        }).then(function () {
                session.call('iot.serial.get').then(
                    function (result) {
                        $('#display').htmlresult);
                    }
                );
            }
        );
    };
    connection.open();
});
```

And thats all. The source code is available in my github [account](https://github.com/gonzalo123/arduino2websockets). You also can see a demo of the working prototype here

[![youtube](https://img.youtube.com/vi/5R82chk01Rs/0.jpg)](https://www.youtube.com/watch?v=5R82chk01Rs)

