---
layout: post
title: "NFC tag reader with Raspberry Pi"
date: "2017-09-04"
categories: 
  - "python"
  - "technology"
tags: 
  - "iot"
  - "mfrc522"
  - "nfc"
  - "raspberry-pi"
coverImage: "112095.png"
---

In [another post](http://gonzalo123.com/2017/06/12/nfc-tag-reader-with-arduino/) we spoke about NFC tag readers and Arduino. Today I'll do the same but with a Raspberry Pi. Why? More or less everything we can do with an Arduino board we can do it also with a Raspberry Pi (and viceversa). Sometimes Arduino is to much low level for me. For example if we want to connect an Arduino to the LAN we need to set up mac address by hand. We can do it but this operation is trivial with Raspberry Pi and Python. We can connect our Arduino to a PostgreSQL Database, but it's not pretty straightforward. Mi background is also better with Python than C++, so I feel more confortable working with Raspberry Pi. I'm not saying that RPi is better than Arduino. With Arduino for example we don't need to worry about start proceses, reboots and those kind of thing stuff that we need to worry about with computers. Arduino a Raspberry Pi are different tools. Sometimes it will be better one and sometimes the other.

So let's start connecting our [RFID/NFC Sensor MFRC522](http://playground.arduino.cc/Learning/MFRC522) to our Raspberry Py 3 The wiring:

- RC522 VCC > RP 3V3
- RC522 RST > RPGPIO25
- RC522 GND > RP Ground
- RC522 MISO > RPGPIO9 (MISO)
- RC522 MOSI > RPGPIO10 (MOSO)
- RC522 SCK > RPGPIO11 (SCLK)
- RC522 NSS > RPGPIO8 (CE0)
- RC522 IRQ > RPNone

I will a Python port of the example code for the NFC module MF522-AN thank to [mxgxw](https://github.com/mxgxw/MFRC522-python)

I'm going to use two Python Scripts. One to control NFC reader

```python
import RPi.GPIO as gpio
import MFRC522
import sys
import time
 
MIFAREReader = MFRC522.MFRC522()
GREEN = 11
RED = 13
YELLOW = 15
SERVO = 12
 
gpio.setup(GREEN, gpio.OUT, initial=gpio.LOW)
gpio.setup(RED, gpio.OUT, initial=gpio.LOW)
gpio.setup(YELLOW, gpio.OUT, initial=gpio.LOW)
gpio.setup(SERVO, gpio.OUT)
p = gpio.PWM(SERVO, 50)
 
good = [211, 200, 106, 217, 168]
 
def servoInit():
    print "servoInit"
    p.start(7.5)
 
def servoOn():
    print "servoOn"
    p.ChangeDutyCycle(4.5)
 
def servoNone():
    print "servoOn"
    p.ChangeDutyCycle(7.5)
 
def servoOff():
    print "servoOff"
    p.ChangeDutyCycle(10.5)
 
def clean():
    gpio.output(GREEN, False)
    gpio.output(RED, False)
    gpio.output(YELLOW, False)
 
def main():
    servoInit()
    while 1:
        (status, TagType) = MIFAREReader.MFRC522_Request(MIFAREReader.PICC_REQIDL)
        if status == MIFAREReader.MI_OK:
            (status, backData) = MIFAREReader.MFRC522_Anticoll()
            gpio.output(YELLOW, True)
            if status == MIFAREReader.MI_OK:
                mac = []
                for x in backData[0:-1]:
                    mac.append(hex(x).split('x')[1].upper())
                print ":".join(mac)
                if good == backData:
                    servoOn()
                    gpio.output(GREEN, True)
                    time.sleep(0.5)
                    servoNone()
                else:
                    gpio.output(RED, True)
                    servoOff()
                    time.sleep(0.5)
                    servoNone()
                time.sleep(1)
                gpio.output(YELLOW, False)
                gpio.output(RED, False)
 
if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print 'Interrupted'
        clean()
        MIFAREReader.GPIO_CLEEN()
        sys.exit(0)
```

And another one to control push button. I use this second script only to see how to use different processes. 

```python
import RPi.GPIO as gpio
import time
 
gpio.setwarnings(False)
gpio.setmode(gpio.BOARD)
BUTTON = 40
GREEN = 11
RED = 13
YELLOW = 15
 
gpio.setup(GREEN, gpio.OUT)
gpio.setup(RED, gpio.OUT)
gpio.setup(YELLOW, gpio.OUT)
 
gpio.setup(BUTTON, gpio.IN, pull_up_down=gpio.PUD_DOWN)
gpio.add_event_detect(BUTTON, gpio.RISING)
def leds(status):
    gpio.output(YELLOW, status)
    gpio.output(GREEN, status)
    gpio.output(RED, status)
 
def buttonCallback(pin):
    if gpio.input(pin) == 1:
        print "STOP"
        leds(True)
        time.sleep(0.2)
        leds(False)
        time.sleep(0.2)
        leds(True)
        time.sleep(0.2)
        leds(False)
 
gpio.add_event_callback(BUTTON, buttonCallback)
while 1:
    pass
```

Here a video with a working example (I've also put a servo and and three leds only because it looks good :))

[![youtube](https://img.youtube.com/vi/IUGeVg7DE9Q/0.jpg)](https://www.youtube.com/watch?v=IUGeVg7DE9Q)

Code in my github [account](https://github.com/gonzalo123/RPi_nfc)
