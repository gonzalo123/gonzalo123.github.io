---
layout: post
title: "Control humidity with a Raspberry Pi and IoT devices"
date: "2017-04-03"
categories: 
  - "node-js"
  - "python"
  - "technology"
tags: 
  - "btle"
  - "iot"
  - "node"
  - "raspberry-pi"
---

I've got a [Wemo switch](http://www.belkin.com/es/F7C029-Belkin/p/P-F7C029/) and a [BeeWi](http://www.bee-wi.com/bbw200,us,4,BBW200-A1.cfm) temperature/humidity sensor. I've use them in previous [projects](https://gonzalo123.com/2016/08/29/home-automation-pet-project-playing-with-iot-temperature-sensors-fans-and-telegram-bots/). Today I want a control humidity level in a room. The idea is switch on/off a dehumidifier (plugged to Wemo switch) depending on the humidity (from BeeWi sensor). Let's start.

I've got one script (node) that reads humidity from the sensor (via BTLE) 

```javascript
#!/usr/bin/env node
noble = require('noble');
 
var status = false;
var address = process.argv[2];
 
if (!address) {
    console.log('Usage "./reader.py <sensor mac address>"');
    process.exit();
}
 
function hexToInt(hex) {
    var num, maxVal;
    if (hex.length % 2 !== 0) {
        hex = "0" + hex;
    }
    num = parseInt(hex, 16);
    maxVal = Math.pow(2, hex.length / 2 * 8);
    if (num > maxVal / 2 - 1) {
        num = num - maxVal;
    }
 
    return num;
}
 
noble.on('stateChange', function(state) {
    status = (state === 'poweredOn');
});
 
noble.on('discover', function(peripheral) {
    if (peripheral.address == address) {
        var data = peripheral.advertisement.manufacturerData.toString('hex');
        console.log(Math.min(100,parseInt(data.substr(14, 2),16)));
        noble.stopScanning();
        process.exit();
    }
});
 
noble.on('scanStop', function() {
    noble.stopScanning();
});
 
setTimeout(function() {
    noble.stopScanning();
    noble.startScanning();
}, 3000);
```

Now I've got another script to control the switch. A Python script using [ouimeaux](https://github.com/iancmcc/ouimeaux) library

```python
#!/usr/bin/env python
from ouimeaux.environment import Environment
from subprocess import check_output
import sys
import os
 
threshold = 3
 
def action(switch):
    humidity = int(check_output(["%s/reader.js" % os.path.dirname(sys.argv[0]), sensorMac]))
    if "Switch1" == switch.name:
        botton = expected - threshold
        isOn = False if switch.get_state() == 0 else True
        log = ""
 
        if isOn and humidity < botton:
            switch.basicevent.SetBinaryState(BinaryState=0)
            log = "humidity < %s Switch to OFF" % botton
        elif not isOn and humidity > expected:
            switch.basicevent.SetBinaryState(BinaryState=1)
            log = "humidity > %s Switch to ON" % expected
 
        print "Humidity: %s Switch is OK (%s) %s" % (humidity, 'On' if isOn else 'Off', log)
 
if __name__ == '__main__':
    try:
        sensorMac = sys.argv[1]
        mySwitch = sys.argv[2]
        expected = int(sys.argv[3])
    except:
        print 'Usage "./dehumidifier.py <sensorMac> <switch name> <expected humidity>"'
        sys.exit()
 
    env = Environment(action)
    env.start()
    env.discover(seconds=3)
```

And that’s all. Now I only need to configure my Raspberry Pi’s crontab and run the script each minute

```
*/1 * * * *     /mnt/media/projects/hum/dehumidifier.py ff:ff:ff:ff:ff:ff Switch1 50
```

Project is available in my [github account](https://github.com/gonzalo123/humidity).

Nowadays I'm involved with Arduino and iot, so I wand to do something similar with cheaper Arduino stuff.
