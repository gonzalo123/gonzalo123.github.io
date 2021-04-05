---
layout: post
title: "Creating QR codes with PHP and Silex"
date: "2013-09-02"
tags: 
  - "php"
  - "symfony"
  - "technology"
  - "qr"
  - "qr-code-generator"
  - "silex"
---

Today we're going to play with QR codes and how to use them within a Silex application using one Service Provider. First we need a QR code generator. If we find in Packagist we can see various libraries. We are going to use the library: [endroid/qrcode](https://packagist.org/packages/endroid/qrcode).

We are not going to modify endroid/qrcode, because of that we will create a wrapper. This wrapper will receive in the constructor one instance of endroid/qrcode. It's responsability will be to take one QrCode object and generate a Symfony\\Component\\HttpFoundation\\Response with our QR code and the properly headers. Here you can see the unit tests of our [QrWrapper](https://github.com/gonzalo123/qrserviceprovider/blob/master/lib/G/QrWrapper.php):

```php
use Symfony\Component\HttpFoundation\Response;
use G\QrWrapper;
 
class QrWrapperTest extends PHPUnit_Framework_TestCase
{
    public function testObjectInit()
    {
        $qrCode = $this->getMockBuilder('Endroid\QrCode\QrCode')
                ->disableOriginalConstructor()
                ->getMock();
 
        $wrapper = new QrWrapper($qrCode);
 
        $this->assertInstanceOf('G\QrWrapper', $wrapper);
    }
 
    public function testGetResponseWithDefaultParameters()
    {
        $qrCode = $this->getMockBuilder('Endroid\QrCode\QrCode')
                ->disableOriginalConstructor()
                ->getMock();
 
        $qrCode->expects($this->any())->method('get')->will($this->returnValue("hello"));
        $wrapper = new QrWrapper($qrCode);
 
        $response = $wrapper->getResponse();
 
        $this->assertInstanceOf('Symfony\Component\HttpFoundation\Response', $response);
        $this->assertEquals("hello", $response->getContent());
        $this->assertEquals("image/png", $response->headers->get('Content-Type'));
    }
 
    public function testGetResponseForJpg()
    {
        $qrCode = $this->getMockBuilder('Endroid\QrCode\QrCode')
                ->disableOriginalConstructor()
                ->getMock();
 
        $qrCode->expects($this->any())->method('get')->will($this->returnValue("hello"));
        $wrapper = new QrWrapper($qrCode);
        $wrapper->setImageType('jpg');
 
        $response = $wrapper->getResponse();
 
        $this->assertInstanceOf('Symfony\Component\HttpFoundation\Response', $response);
        $this->assertEquals("hello", $response->getContent());
        $this->assertEquals("image/jpeg", $response->headers->get('Content-Type'));
    }
 
    public function testGetResponseForJpeg()
    {
        $qrCode = $this->getMockBuilder('Endroid\QrCode\QrCode')
                ->disableOriginalConstructor()
                ->getMock();
 
        $qrCode->expects($this->any())->method('get')->will($this->returnValue("hello"));
        $wrapper = new QrWrapper($qrCode);
        $wrapper->setImageType('jpeg');
 
        $response = $wrapper->getResponse();
 
        $this->assertInstanceOf('Symfony\Component\HttpFoundation\Response', $response);
        $this->assertEquals("hello", $response->getContent());
        $this->assertEquals("image/jpeg", $response->headers->get('Content-Type'));
    }
 
    public function testReusingResponse()
    {
        $qrCode = $this->getMockBuilder('Endroid\QrCode\QrCode')
                ->disableOriginalConstructor()
                ->getMock();
 
        $qrCode->expects($this->any())->method('get')->will($this->returnValue("hello"));
        $wrapper = new QrWrapper($qrCode);
 
        $response = new Response('foo');
        $response->headers->set('xxx', 'gonzalo');
 
        $response = $wrapper->getResponse($response);
 
        $this->assertEquals("hello", $response->getContent());
        $this->assertEquals("image/png", $response->headers->get('Content-Type'));
        $this->assertEquals("gonzalo", $response->headers->get('xxx'));
    }
}
```

Now we will create the ServiceProvider. We only need to implement ServiceProviderInterface 

```php
use Silex\Application;
use Silex\ServiceProviderInterface;
use Endroid\QrCode\QrCode;
 
class QrServiceProvider implements ServiceProviderInterface
{
    public function register(Application $app)
    {
        $app['qrCode'] = $app->protect(function ($text, $size = null) use ($app) {
            $default = $app['qr.defaults'];
 
            $qr = new QrWrapper(new QrCode());
            $qr->setText($text);
            $qr->setPadding($default['padding']);
            $qr->setSize(is_null($size) ? $default['size'] : $size);
            $qr->setImageType($default['imageType']);
 
            return $qr;
        });
    }
 
    public function boot(Application $app)
    {
    }
}
```

And that's all. Now we can use our service provider within one Silex Application: 

```php
use Silex\Application;
use G\QrServiceProvider;
 
$app = new Application();
 
$app->register(new QrServiceProvider(), [
    'qr.defaults' => [
        'padding'   => 5, // default: 0
        'size'      => 200,
        'imageType' => 'png', // png, gif, jpeg, wbmp (default: png)
    ]
]);
 
$app->get("/qr/base64/{text}", function($text) use ($app) {
    return $app['qrCode'](base64_decode($text))->getResponse();
});
 
$app->get("/qr/{text}", function($text) use ($app) {
    return $app['qrCode']($text)->getResponse();
});
 
$app->run();
```

You can fetch the full code in [github](https://github.com/gonzalo123/qrserviceprovider) and also use it with [composer](https://packagist.org/packages/gonzalo123/qrserviceprovider)
