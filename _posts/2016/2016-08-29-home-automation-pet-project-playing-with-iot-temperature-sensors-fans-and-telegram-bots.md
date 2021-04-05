---
layout: post
title: "Home automation pet project. Playing with IoT, temperature sensors, fans and Telegram bots"
date: "2016-08-29"
tags: 
  - "js"
  - "node-js"
  - "python"
  - "technology"
  - "automation"
  - "iot"
  - "node"
  - "raspberry-pi"
---

Summer holidays are over. Besides my bush walks I've been also hacking a little bit with one idea that I had in mind. Summer means high temperatures and I wanted to control my fan. For example turn on the fan when temperature is over a threshold. I can do it using an Arduino board and a temperature sensor, but I don't have the one Arduino board. I have several devices. For example a [Wemo switch](http://www.belkin.com/es/F7C029-Belkin/p/P-F7C029/). With this device connected to my Wifi network I can switch on and off my fan remotely from my mobile phone (using its android app) or even from my Pebble watch using the [API](https://pypi.python.org/pypi/ouimeaux). I also have a [BeeWi](http://www.bee-wi.com/bbw200,us,4,BBW200-A1.cfm) temperature/humidity sensor. It's a BTLE device. It comes with its own app for android, but there's also a API. Yes. I known that one Arduino board with a couple of sensors can be cheaper than one of this devices, but when I'm a shop and I've got one of this devices in my hands I cannot resist.

I also have a new Raspberry pi 3. I've recently upgraded my home multimedia server from a rpi2 to the new rpi3. Basically I use it as [multimedia](https://www.plex.tv/) server and now also as [retro console](https://retropie.org.uk/). This new rpi3 has Bluetooth so I wanted to do something with it. Read temperature from the Bluetooth sensor sounds good so I started to hack a little bit.

I found this [post](https://www.raspberrypi.org/forums/viewtopic.php?f=37&t=139848). I started working with Python. The script almost works but it uses Bluetooth connection and as someone said in the comments it uses a lot of battery. So I switched to a BTLE version. I found a simple node library to connect BTLE devices called [noble](https://github.com/sandeepmistry/noble), really simple to use. In one afternoon I had one small script ready. The idea was put this script in my RP3's crontab, and scan the temperature each minute (via noble) and if the temperature was over a threshold switch on the wemo device (via ouimeaux). I also wanted to be informed when my fan is switch on and off. The most easier way to do it was via Telegram (I already knew [telebot](https://github.com/kosmodrey/telebot) library).

```javascript
var noble = require('noble'),
    Wemo = require('wemo-client'),
    TeleBot = require('telebot'),
    fs = require('fs'),
    beeWiData,
    wemo,
    threshold,
    address,
    bot,
    chatId,
    wemoDevice,
    configuration,
    confPath;
 
if (process.argv.length <= 2) {
    console.log("Usage: " + __filename + " conf.json");
    process.exit(-1);
}
 
confPath = process.argv[2];
try {
    configuration = JSON.parse(
        fs.readFileSync(process.argv[2])
    );
} catch (e) {
    console.log("configuration file not valid");
    process.exit(-1);
}
 
bot = new TeleBot(configuration.telegramBotAPIKey);
address = configuration.beeWiAddress;
threshold = configuration.threshold;
wemoDevice = configuration.wemoDevice;
chatId = configuration.telegramChatId;
 
function persists() {
    configuration.beeWiData = beeWiData;
    fs.writeFileSync(confPath, JSON.stringify(configuration));
}
 
function setSwitchState(state, callback) {
    wemo = new Wemo();
    wemo.discover(function(deviceInfo) {
        if (deviceInfo.friendlyName == wemoDevice) {
            console.log("device found:", deviceInfo.friendlyName, "setting the state to", state);
            var client = wemo.client(deviceInfo);
            client.on('binaryState', function(value) {
                callback();
            });
 
            client.on('statusChange', function(a) {
                console.log("statusChange", a);
            });
            client.setBinaryState(state);
        }
    });
}
 
beeWiData = {temperature: undefined, humidity: undefined, batery: undefined};
 
function hexToInt(hex) {
    if (hex.length % 2 !== 0) {
        hex = "0" + hex;
    }
    var num = parseInt(hex, 16);
    var maxVal = Math.pow(2, hex.length / 2 * 8);
    if (num > maxVal / 2 - 1) {
        num = num - maxVal;
    }
    return num;
}
 
noble.on('stateChange', function(state) {
    if (state === 'poweredOn') {
        noble.stopScanning();
        noble.startScanning();
    } else {
        noble.stopScanning();
    }
});
 
noble.on('scanStop', function() {
    var message, state;
    if (beeWiData.temperature > threshold) {
        state = 1;
        message = "temperature (" + beeWiData.temperature + ") over threshold (" + threshold + "). Fan ON. Humidity: " + beeWiData.humidity;
    } else {
        message = "temperature (" + beeWiData.temperature + ") under threshold (" + threshold + "). Fan OFF. Humidity: " + beeWiData.humidity;
        state = 0;
    }
    setSwitchState(state, function() {
        if (configuration.beeWiData.hasOwnProperty('temperature') && configuration.beeWiData.temperature < threshold && state === 1 || configuration.beeWiData.temperature > threshold && state === 0) {
            console.log("Notify to telegram bot", message);
            bot.sendMessage(chatId, message).then(function() {
                process.exit(0);
            }, function(e) {
                console.error(e);
                process.exit(0);
            });
            persists();
        } else {
            console.log(message);
            persists();
            process.exit(0);
        }
    });
});
 
noble.on('discover', function(peripheral) {
    if (peripheral.address == address) {
        var data = peripheral.advertisement.manufacturerData.toString('hex');
        beeWiData.temperature = parseFloat(hexToInt(data.substr(10, 2)+data.substr(8, 2))/10).toFixed(1);
        beeWiData.humidity = Math.min(100,parseInt(data.substr(14, 2),16));
        beeWiData.batery = parseInt(data.substr(24, 2),16);
        beeWiData.date = new Date();
        noble.stopScanning();
    }
});
 
setTimeout(function() {
    console.error("timeout exceded!");
    process.exit(0);
}, 5000);
```

The script is [here](https://github.com/gonzalo123/fan/).

It works but I wanted to keep on hacking. One Sunday morning I read this [post](https://medium.com/@edwardbenson/how-i-hacked-amazon-s-5-wifi-button-to-track-baby-data-794214b0bdd8#.brqlow60y). I don't have an amazon button, but I wanted to do something similar. I started to play with [scapy](https://github.com/secdev/scapy) library sniffing ARP packets in my home network. I realize that I can detect when my Kindle connects to the network, my tv, or even my mobile phone. Then I had one I idea: Detect when my mobile phone connects to my wifi. My mobile phone connects to my wifi before I enter in my house so my idea was simple: Detect when I'm close to my home's door and send me a telegram message saying "Wellcome home" in addition to the temperature inside my house at this moment.

```python
#!/usr/bin/env python
 
import sys
from scapy.all import *
import telebot
import gearman
import json
from StringIO import StringIO
 
BUFFER_SIZE = 1024
 
try:
    with open(sys.argv[1]) as data_file:
        data = json.load(data_file)
        myPhone = data['myPhone']
        routerIP = data['routerIP']
        TOKEN = data['telegramBotAPIKey']
        chatID = data['telegramChatId']
        gearmanServer = data['gearmanServer']
except:
    print("Unexpected error:", sys.exc_info()[0])
    raise
 
def getSensorData():
    gm_client = gearman.GearmanClient([gearmanServer])
    completed_job_request = gm_client.submit_job("temp", '')
    io = StringIO(completed_job_request.result)
 
    return json.load(io)
 
tb = telebot.TeleBot(TOKEN)
 
def arp_display(pkt):
    if pkt[ARP].op == 1 and pkt[ARP].hwsrc == myPhone and pkt[ARP].pdst == routerIP:
        sensorData = getSensorData()
        message = "Wellcome home Gonzalo! Temperature: %s humidity: %s" % (sensorData['temperature'], sensorData['humidity'])
        tb.send_message(chatID, message)
        print message
 
print sniff(prn=arp_display, filter='arp', store=0)
```

I have one node script to read temperature and one Python script to sniff my network. I can find how to read temperature from Python and use only one script but I was lazy (remember that I was on holiday) so I turned the node script that reads temperature into a [gearman](http://gearman.org/) worker.

```javascript
var noble = require('noble'),
    fs = require('fs'),
    Gearman = require('node-gearman'),
    beeWiData,
    address,
    bot,
    configuration,
    confPath,
    status,
    callback;
 
var gearman = new Gearman();
 
if (process.argv.length <= 2) {
    console.log("Usage: " + __filename + " conf.json");
    process.exit(-1);
}
 
confPath = process.argv[2];
try {
    configuration = JSON.parse(
        fs.readFileSync(process.argv[2])
    );
} catch (e) {
    console.log("configuration file not valid", e);
    process.exit(-1);
}
 
address = configuration.beeWiAddress;
delay = configuration.tempServerDelayMinutes * 60 * 1000;
tcpPort = configuration.tempServerPort;
 
beeWiData = {};
 
function hexToInt(hex) {
    if (hex.length % 2 !== 0) {
        hex = "0" + hex;
    }
    var num = parseInt(hex, 16);
    var maxVal = Math.pow(2, hex.length / 2 * 8);
    if (num > maxVal / 2 - 1) {
        num = num - maxVal;
    }
    return num;
}
 
noble.on('stateChange', function(state) {
    if (state === 'poweredOn') {
        console.log("stateChange:poweredOn");
        status = true;
    } else {
        status = false;
    }
});
 
noble.on('discover', function(peripheral) {
    if (peripheral.address == address) {
        var data = peripheral.advertisement.manufacturerData.toString('hex');
        beeWiData.temperature = parseFloat(hexToInt(data.substr(10, 2)+data.substr(8, 2))/10).toFixed(1);
        beeWiData.humidity = Math.min(100,parseInt(data.substr(14, 2),16));
        beeWiData.batery = parseInt(data.substr(24, 2),16);
        beeWiData.date = new Date();
        noble.stopScanning();
    }
});
 
noble.on('scanStop', function() {
    console.log(beeWiData);
    noble.stopScanning();
    callback();
});
 
var worker;
 
function workerCallback(payload, worker) {
    callback = function() {
        worker.end(JSON.stringify(beeWiData));
    }
 
    beeWiData = {temperature: undefined, humidity: undefined, batery: undefined};
 
    if (status) {
        noble.stopScanning();
        noble.startScanning();
    } else {
        setInterval(function() {
            workerCallback(payload, worker);
        }, 1000);
    }
}
 
gearman.registerWorker("temp", workerCallback);
```

Now I only need to call this worker from my Python sniffer and thats all.

I wanted to play a little bit. I also wanted to ask the temperature on demand. Since I was using Telegram I had an idea. Create a Telegram bot running in my RP3. And that's my summer pet project. Basically it has three parts:

**worker.js** 
It's a gearman worker. It reads temperature and humidity from my BeeWi sensor via BTLE

**bot.py** 
It's a Telegram bot with the following commands available:

> /switchInfo: get switch info /switchOFF: switch OFF the switch /help: Gives you information about the available commands /temp: Get temperature /switchON: switch ON the switch

**sniff.py** 
It's just a ARP sniffer. It detects when I'm close to my home and sends me a message via Telegram with the temperature. It detects when my mobile phone sends a ARP package to my router (aka when I connect to my Wifi). It happens before I enter in my house, so the Telegram message arrives before I put the key in the door :)

I run al my scripts in my Raspberry Pi3. To ensure all scripts are up an running I use [supervisor](http://supervisord.org/)

All the scripts are available in my github [account](https://github.com/gonzalo123/home-automation)
