---
layout: post
title: "Arduino and Raspberry Pi working together"
date: "2017-05-08"
categories: 
  - "arduino"
  - "python"
  - "technology"
tags: 
  - "iot"
  - "raspberry-pi"
coverImage: "112098.png"
---

Basically everything we can do with Arduino it can be done also with a Raspberry Pi (an viceversa). There're things that they're easy to do with Arduino (connect sensors for example). But another things (such as work with REST servers, databases, ...) are "complicated" with Arduino and C++ (they are possible but require a lot of low level operations) and pretty straightforward with Raspberry Pi and Python (at least for me and because of my background)

With this small project I want to use an Arduino board and Raspberry Pi working together. The idea is blink two LEDs. One (green one) will be controlled by Raspberry Pi directly via GPIO and another one (red one) will be controlled by Arduino board. Raspberry Pi will be the "brain" of the project and will tell to Arduino board when turn on/off it's led. Let's show you the code.

```python
import RPi.GPIO as gpio
import serial
import time
import sys
import os
 
def main():
    gpio.setmode(gpio.BOARD)
    gpio.setup(12, gpio.OUT)
 
    s = serial.Serial('/dev/ttyACM0', 9600)
    status = False
 
    while 1:
        gpio.output(12, status)
        status = not status
        print status
        s.write("1\n" if status else "0\n")
        time.sleep(1)
 
if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print 'Interrupted'
        gpio.cleanup()
        try:
            sys.exit(0)
        except SystemExit:
            os._exit(0)
```

As we can see the script is a simple loop and blink led (using pin 12) with one interval of one second. Our Arduino board is connected directly to the Raspberry Pi via USB cable and we send commands via serial interface.

Finally the Arduino program: 

```c
#define LED  11
 
String serialData = "";
boolean onSerialRead = false; 
 
void setup() {
  // initialize serial:
  Serial.begin(9600);
  serialData.reserve(200);
}
 
void procesSerialData() {
  Serial.print("Data " + serialData);
  if (serialData == "1") {    
    Serial.println(" LED ON");
    digitalWrite(LED, HIGH);
  } else {
    Serial.println(" LED OFF");
    digitalWrite(LED, LOW);
  }
  serialData = "";
  onSerialRead = false;
}
 
void loop() {
  if (onSerialRead) {
    procesSerialData();
  }
}
 
void serialEvent() {
  while (Serial.available()) {
    char inChar = (char)Serial.read();
    if (inChar == '\n') {
      onSerialRead = true;
    } else {
      serialData += inChar;
    }
  }
}
```

Here our Arduino Board is listening to serial interface (with serialEvent) and each time we receive "\\n" the main loop will turn on/off the led depending on value (1 - On, 0 - Off)

We can use I2C and another ways to connect Arduino and Raspberry Pi but in this example we're using the simplest way to do it: A USB cable. We only need a A/B USB cable. We don't need any other extra hardware (such as resistors) and the software part is pretty straightforward also.

Hardware:

- Arduino UNO
- Raspberry Pi 3
- Two LEDs and two resistors

[![youtube](https://img.youtube.com/vi/jlr8P74OdUk/0.jpg)](https://www.youtube.com/watch?v=jlr8P74OdUk)

Code in my github [account](https://github.com/gonzalo123/arduino_RPi_together/)
