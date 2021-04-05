---
layout: post
title: "Handling dates with PHP"
date: "2012-10-01"
tags: 
  - "php"
  - "phpunit"
  - "technology"
  - "datetime"
---

I've seen a lot of newbies (and not newbies) having problems handling dates in PHP (and even with SQL and another languages). When I see someone having problems with dates, I always ask the same question. I type in a text editor "27/11/2012" and I ask him: What is it? If your answer is "This is a date" you should continue reading the post :)

"27/11/2012" in a text editor is not a date. It's a string representing a date (my birthday this year, indeed :) ). We also must take into account that we cannot show a "raw" date. We only can apply a mask to a date if we want to show it. Because of that dates are normally stored internally as UNIX [timestams](http://en.wikipedia.org/wiki/Timestamps) or something similar.You can say that when we perform one SELECT statement with a date field we don't see one timestamp. That's because the tool that you are using to show the date field applies one mask to your date. The default mask can be also a problem. "01/02/2012" represents 01/February/2012 or 02/January/2012? We don't know. Depends on the mask that we are using.

When we work with PHP is very usual to use dates. Sometimes we need to pass them to SQL queries, perform operations (such as add days, diffs, compare, ...). When I started with PHP 10 years ago we use to use [mktime()](http://php.net/manual/function.mktime.php), [date()](http://php.net/manual/function.date.php), and insane operations with split/join with strings. That's the past. Now we have [DateTime](http://php.net/manual/book.datetime.php).

It's pretty straightforward to use it. We can use one static factory to create DateTime objects

```php
$date = DateTime::createFromFormat('d/m/Y', '27/11/2012');
```

Now $date is a variable that stores a DateTime object. If we want to show it, we need to apply one format. For example: 

```php
$date = DateTime::createFromFormat('d/m/Y', '27/11/2012');
echo $date->format('m/d/Y');  // outputs 11/27/2012
```

I'm not going to create a tutorial of DateTime here. It's very good described (as always) within PHP official [documentation](http://php.net/manual/class.datetime.php). There is something that I love with DateTime objects also:

```php
$date1 = DateTime::createFromFormat('d/m/Y', '27/11/2012');
$date2 = DateTime::createFromFormat('d/m/Y', '27/11/2011');
 
if ($date1 > $date2) {
    echo "It works!"
}
```

In my last projects when I need to use Dates I like to cast the variables as DateTime objects. Let me show you a little example: 

```php
<?php
 
class Dummy
{
    /** @var DateTime */
    private $date;
    private $defaultFormat = 'd/m/Y';
 
    public function setDate(DateTime $date)
    {
        $this->date = $date;
    }
 
    public function getDate()
    {
        return $this->date;
    }
 
    public function getFormattedDate($format = NULL)
    {
        return $this->date->format($this->getFormat($format));
    }
 
    public function setFormattedDate($formatedDate, $format = NULL)
    {
        $this->date = DateTime::createFromFormat($this->getFormat($format), $formatedDate);
    }
 
    public function setDefaultFormat($format)
    {
        $this->defaultFormat = $format;
    }
 
    private function getFormat($format)
    {
        return is_null($format) ? $this->defaultFormat : $format;
    }
}
```

But maybe, as always, the best documentation are the Unit Tests 

```php
<?php
 
class DummyTest extends \PHPUnit_Framework_TestCase
{
    public function testDate()
    {
        $date  = DateTime::createFromFormat('d/m/Y', '1/2/2012');
        $dummy = new Dummy();
        $dummy->setDate($date);
        $this->assertEquals($date, $dummy->getDate());
    }
 
    public function testDateWithAlternativeSetter()
    {
        $dummy = new Dummy();
        $dummy->setFormattedDate('1/2/2012', 'd/m/Y');
        $this->assertEquals(DateTime::createFromFormat('d/m/Y', '1/2/2012'), $dummy->getDate());
    }
 
    public function testDateWithAlternativeSetterWithDeffaultFormatSet()
    {
        $dummy = new Dummy();
        $dummy->setDefaultFormat('d/m/Y');
        $dummy->setFormattedDate('1/2/2012');
        $this->assertEquals(DateTime::createFromFormat('d/m/Y', '1/2/2012'), $dummy->getDate());
    }
 
    public function testFormatGetterWithDefaultFormat()
    {
        $dummy = new Dummy();
        $dummy->setFormattedDate('1/2/2012');
        $this->assertEquals('2012', $dummy->getDate()->format('Y'));
    }
     
    public function testFormatGetter()
    {
        $dummy = new Dummy();
        $dummy->setFormattedDate('1/2/2012');
        $this->assertEquals('2012', $dummy->getFormattedDate('Y'));
    }
 
    public function testDiffDates()
    {
        $dummy = new Dummy();
        $dummy->setFormattedDate('1/2/2012');
        $date = DateTime::createFromFormat('d/m/Y', '1/2/2011');
 
        $this->assertTrue($date < $dummy->getDate());
    }
 
    public function testModifyDates()
    {
        // leap-years
        $dummy = new Dummy();
        $dummy->setFormattedDate('28/02/1908');
        $this->assertEquals('29/02/1908', $dummy->getDate()->modify("+1 day")->format('d/m/Y'));
 
        // Non leap-years
        $dummy = new Dummy();
        $dummy->setFormattedDate('28/02/2001');
        $this->assertEquals('01/03/2001', $dummy->getDate()->modify("+1 day")->format('d/m/Y'));
    }
}
```

What do you think?
