---
layout: post
title: "Arduino and Raspberry Pi working together. Part 2 (now with i2c)"
date: "2017-05-22"
categories: 
  - "arduino"
  - "python"
  - "technology"
tags: 
  - "i2c"
  - "iot"
  - "raspberry-pi"
coverImage: "112101.png"
---

The easiest way to connect our Arduino board to our Raspberry Py is using the USB [cable](http://gonzalo123.com/2017/05/08/arduino-and-raspberry-pi-working-together/), but sometimes this communication is a nightmare, especially because there isn't any clock signal to synchronize our devices and we must rely on the bitrate. There're different ways to connect our Arduino and our Raspberry Py such as I2C, SPI and serial over GPIO. Today we're going to speak about I2C, especially because it's pretty straightforward if we take care with a couple of things. Let's start.

**I2C** uses two lines **SDA** (data) and **SCL** (clock), in addition to **GND** (ground). SDA is bidirectional so we need to ensure, in one way or another, who is sending data (master or slave). With I2C only master can start communications and also master controls the clock signal. Each device has a 7bit direction so we can connect 128 devices to the same bus.

If we want to connect Arduino board and Raspberry Pi we must ensure that Raspberry Pi is the master. That's because Arduino works with 5V and Raspberry Pi with 3.3V. That means that we need to use pull-up resistors if we don't want destroy our Raspberry Pi. But Raspberry Pi has 1k8 ohms resistors to the 3.3 votl power rail, so we can connect both devices (if we connect other i2c devices to the bus they must have their pull-up resistors removed)

Thats all we need to connect our Raspberry pi to our Arduino board.

- RPi SDA to Arduino analog 4
- RPi SCL to Arduino analog 5
- RPi GND to Arduino GND

![](/assets/images/untitled_sketch_fzz_-_fritzing_-__breadboard_view_.png)

Now we are going to build a simple prototype. Raspberry Pi will blink one led (GPIO17) each second and also will send a message (via I2C) to Arduino to blink another led. That's the Python part

```python
import RPi.GPIO as gpio
import smbus
import time
import sys
 
bus = smbus.SMBus(1)
address = 0x04
 
def main():
    gpio.setmode(gpio.BCM)
    gpio.setup(17, gpio.OUT)
    status = False
    while 1:
        gpio.output(17, status)
        status = not status
        bus.write_byte(address, 1 if status else 0)
        print "Arduino answer to RPI:", bus.read_byte(address)
        time.sleep(1)
 
 
if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print 'Interrupted'
        gpio.cleanup()
        sys.exit(0)
```

And finally the Arduino program. Arduino also answers to Raspberry Pi with the value that it's been sent, and Raspberry Pi will log the answer within console.

```c
#include <Wire.h>
 
#define SLAVE_ADDRESS 0x04
#define LED  13
 
int number = 0;
 
void setup() {
  pinMode(LED, OUTPUT);
  Serial.begin(9600);
  Wire.begin(SLAVE_ADDRESS);
  Wire.onReceive(receiveData);
  Wire.onRequest(sendData);
 
  Serial.println("Ready!");
}
 
void loop() {
  delay(100);
}
 
void receiveData(int byteCount) {
  Serial.print("receiveData");
  while (Wire.available()) {
    number = Wire.read();
    Serial.print("data received: ");
    Serial.println(number);
 
    if (number == 1) {
      Serial.println(" LED ON");
      digitalWrite(LED, HIGH);
    } else {
      Serial.println(" LED OFF");
      digitalWrite(LED, LOW);
    }
  }
}
 
void sendData() {
  Wire.write(number);
}
```

Hardware:

- Arduino UNO
- Raspberry Pi
- Two LEDs and two resistors


[![youtube](https://img.youtube.com/vi/EMunlSg77DA/0.jpg)](https://www.youtube.com/watch?v=EMunlSg77DA)

Code available in my [github](https://github.com/gonzalo123/arduino_RPi_i2c)
