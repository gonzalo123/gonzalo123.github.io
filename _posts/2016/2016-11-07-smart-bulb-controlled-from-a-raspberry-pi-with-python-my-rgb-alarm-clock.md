---
layout: post
title: "Smart bulb controlled from a Raspberry Pi with Python. My RGB alarm clock"
date: "2016-11-07"
tags: 
  - "python"
  - "technology"
  - "telegram"
  - "bot"
  - "cron"
  - "iot"
  - "raspberry-pi"
  - "smart-bulb"
---

I've got a BeeWi Smart LED Color [Bulb](http://www.bee-wi.com/bbl207,us,4,BBL207-A1.cfm). I must admit I cannot resist to buy those kind of devices :).

I can switch on/off the bulb and change the color using its Mobile App, but it's not fun. I want to play a little bit with the bulb. My idea is the following one: First switch on the bulb in the mornint and set up the bulb color (Blue for example). Then change bulb color depending on my morning routine. And finally switch the bulb off. Now with this bulb's color I know if my morning routine is on-time, just looking at the bulb's color. For example if the bulb is red and I'm still having breakfast probably I'm late.

The prototype is very simple. The bulb has a bluetooth interface and I've found a [python script](https://www.raspberrypi.org/forums/viewtopic.php?f=37&t=117729) to control the bulb. I've changed a little bit this script to adapt it to my [needs](https://github.com/gonzalo123/home-automation/blob/master/bulb.py).

Now I only need to set up the crontab within my Raspberry Pi to trigger the script and switch on/off the bulb and change the RGB color.

for example: 

```commandline
# switch on the bulb
/usr/bin/python /mnt/media/projects/iot/bulb.py /mnt/media/projects/iot/conf.json on
# set bulb's color to green
/usr/bin/python /mnt/media/projects/iot/bulb.py /mnt/media/projects/iot/conf.json colour 999900

```

In another post we play with Telegram bots to read [temperature](https://gonzalo123.com/2016/08/29/home-automation-pet-project-playing-with-iot-temperature-sensors-fans-and-telegram-bots/). Now I've adapted also my bot to switch on/off and change color of the [bulb](https://github.com/gonzalo123/home-automation).

Now I've got another toy in my desk. One arduino board. I'm sure I will enjoy a lot :)
