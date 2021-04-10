---
layout: post
title: "Howto Send/Read SMSs using a GSM modem, AT+ commands and PHP"
date: "2011-03-21"
tags: 
  - "php"
  - "technology"
  - "gsm"
  - "sms"
---

I need to send/read SMS with PHP. To do that, I need a [GSM modem](http://www.google.com/images?q=GSM+modem&biw=1920&bih=967). There are few of them if we google a little bit. GSM modems are similar than normal modems. They've got a SIM card and we can do the same things we can do with a mobile phone, but using AT and AT+ commands programmatically. That’s means we can send (and read) SMSs and create scripts to perform those operations. Normally those kind of devices uses a serial interface. So we need to connect our PC/server with a serial cable to the device. That’s a problem. Modern PCs sometimes dont’t have serial ports. Modern GSM modems have a USB port, and even we can use serial/USB converters. I don’t like to connect directly those devices to the PC/server. They must be close. Serial/USB cables cannot be very long (2 or 3 meters). I prefer to connect those devices using serial/ethernet converters. Because of that I will create a dual library. The idea is enable the operations when device is connected via serial cable and also when it’s connected thought a serial/ethernet converter.

The idea is the following one: We are going to create a main class called Sms. It takes in the constructor (via dependency injection) the HTTP wrapper or the serial one (both with the same interface). That means our Sms class will work exactly in the same way with one interface or another.

Let’s start. First we’re going to create a Dummy Mock object, sharing the same interface than the others. The purpose of that is to test the main class (Sms.php)

```php
<?php
class Sms_Dummy implements Sms_Interface
{
    public function deviceOpen()
    {
    }
 
    public function deviceClose()
    {
    }
 
    public function sendMessage($msg)
    {
    }
 
    public function readPort()
    {
        return array("OK", array());
    }
 
    private $_validOutputs = array();
 
    public function setValidOutputs($validOutputs)
    {
        $this->_validOutputs = $validOutputs;
    }
}
```

And we test the code:

```php
<?php
 
require_once('Sms.php');
require_once('Sms/Interface.php');
require_once('Sms/Dummy.php');
 
$pin = 1234;
 
$serial = new Sms_Dummy;
 
if (Sms::factory($serial)->insertPin($pin)
                ->sendSMS(555987654, "test Hi")) {
    echo "SMS sent\n";
} else {
    echo "SMS not Sent\n";
}
```

It works. Our Sms class sends a fake sms using Sms\_Dummy

Now we only need to create Sms\_Serial and Sms\_Http. For Sms\_Serial I’ve use the great library (with a slight modifications) of Rémy Sanchez to read/write serial port in PHP (you can find it [here](http://www.phpclasses.org/package/3679-PHP-Communicate-with-a-serial-port.html)). We can run it with a script similar than the Dummy one, thanks to dependency injection:

```php
require_once('Sms.php');
require_once('Sms/Interface.php');
require_once('Sms/Serial.php');
 
$pin = 1234;
 
try {
    $serial = new Sms_Serial;
    $serial->deviceSet("/dev/ttyS0");
    $serial->confBaudRate(9600);
    $serial->confParity('none');
    $serial->confCharacterLength(8);
 
    $sms = Sms::factory($serial)->insertPin($pin);
 
    if ($sms->sendSMS(555987654, "test Hi")) {
        echo "SMS sent\n";
    } else {
        echo "Sent Error\n";
    }
 
    // Now read inbox
    foreach ($sms->readInbox() as $in) {
        echo"tlfn: {$in['tlfn']} date: {$in['date']} {$in['hour']}\n{$in['msg']}\n";
 
        // now delete sms
        if ($sms->deleteSms($in['id'])) {
            echo "SMS Deleted\n";
        }
    }
} catch (Exception $e) {
    switch ($e->getCode()) {
        case Sms::EXCEPTION_NO_PIN:
            echo "PIN Not set\n";
            break;
        case Sms::EXCEPTION_PIN_ERROR:
            echo "PIN Incorrect\n";
            break;
        case Sms::EXCEPTION_SERVICE_NOT_IMPLEMENTED:
            echo "Service Not implemented\n";
            break;
        default:
            echo $e->getMessage();
    }
}
```

And finaly the Http one. As you can see the the script is the same than the Serial one. the only difference is the class passed to the constructor of the Sms class:

```php
<?php 
require_once('Sms.php'); 
require_once('Sms/Interface.php'); 
require_once('Sms/Http.php'); 
 
$serialEternetConverterIP = '192.168.1.10'; 
$serialEternetConverterPort = 1113; 
$pin = 1234; 
try {
     $sms = Sms::factory(new Sms_Http($serialEternetConverterIP, $serialEternetConverterPort));
     $sms->insertPin($pin);
 
    if ($sms->sendSMS(555987654, "test Hi")) {
        echo "SMS Sent\n";
    } else {
        echo "Sent Error\n";
    }
 
    // Now read inbox
    foreach ($sms->readInbox() as $in) {
        echo"tlfn: {$in['tlfn']} date: {$in['date']} {$in['hour']}\n{$in['msg']}\n";
 
        // now delete sms
        if ($sms->deleteSms($in['id'])) {
            echo "SMS Deleted\n";
        }
    }
} catch (Exception $e) {
    switch ($e->getCode()) {
        case Sms::EXCEPTION_NO_PIN:
            echo "PIN Not set\n";
            break;
        case Sms::EXCEPTION_PIN_ERROR:
            echo "PIN Incorrect\n";
            break;
        case Sms::EXCEPTION_SERVICE_NOT_IMPLEMENTED:
            echo "Service Not implemented\n";
            break;
        default:
            echo $e->getMessage();
    }
}
```

Serial/Ethernet [converters](http://www.google.com/images?q=ethernet+serial+converter&biw=1920&bih=967) normally have different operational modes. They allow you to create a fake serial port on your PC and work exactly in the same way than if you have your device connected with a serial cable. I don’t like this operation mode. I prefer to use the TCP server mode. With the TCP server mode I can read/write from the serial device with the standard socket functions. So if you dive into [Sms\_Http](https://github.com/gonzalo123/gam-sms/blob/master/Sms/Http.php) class you will find TCP socket's functions.

And finally I will do a brief excerpt of AT+ commands used to send /read SMSs:

**AT+CPIN?\\r** : checks if SIM has the pin code. It answers +CPIN: READY or +CPIN: SIM PIN if we need to insert the pin number **AT+CMGS=”\[number\]”\\r\[text\]Control-Z** : to send a SMS (in php we have Control-Z with chr(26)). It returns OK or ERROR **AT+CMGF=1\\r**: set the device to operate in SMS text mode (0=PDU mode and 1=text mode). Returns OK. **AT+CMGL=\\"ALL"\\r** read all the sms stored in the device. We also can use “REC UNREAD” Instead of “ALL”. **AT+CMGD=\[ID\]\\r**: Deletes a SMS from the device

You can download the source code from github [here](https://github.com/gonzalo123/gam-sms)
