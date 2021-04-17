---
layout: post
title: "Pivot tables in PHP"
date: "2010-01-24"
tags: 
  - "php"
categories: 
  - "php"
permalink: /:year/:month/:day/:title/
---

In my work as developer I normally need to transform data from one format to another. Typically our input data is a database and we need to show the database data into a report, datagrid or something similar. It's very typical to use [pivot tables](http://en.wikipedia.org/wiki/Pivot_table). It's not very dificult to handle pivot tables by hand but the work is always the same: groups by, totals, subtotals, totals per row .... Now I want to write a class to pivot tables in PHP with the most common requirements (at least for me). I know we can create pivot tables with SQL. Group by and some other database specific commands like oracle's GROUP BY ROLLUP/CUBE can do the work, but I like clean SQL querys. The business logic must be in PHP and SQL must be as clean as we can.

Let's show you an example. We have the following recordset from a SQL:

```php
$recordset = array(
    array('host' => 1, 'country' => 'fr', 'year' => 2010,
        'month' => 1, 'clicks' => 123, 'users' => 4),
    array('host' => 1, 'country' => 'fr', 'year' => 2010,
        'month' => 2, 'clicks' => 134, 'users' => 5),
    array('host' => 1, 'country' => 'fr', 'year' => 2010,
        'month' => 3, 'clicks' => 341, 'users' => 2),
    array('host' => 1, 'country' => 'es', 'year' => 2010,
        'month' => 1, 'clicks' => 113, 'users' => 4),
    array('host' => 1, 'country' => 'es', 'year' => 2010,
        'month' => 2, 'clicks' => 234, 'users' => 5),
    array('host' => 1, 'country' => 'es', 'year' => 2010,
        'month' => 3, 'clicks' => 421, 'users' => 2),
    array('host' => 1, 'country' => 'es', 'year' => 2010,
        'month' => 4, 'clicks' => 22,  'users' => 3),
    array('host' => 2, 'country' => 'es', 'year' => 2010,
        'month' => 1, 'clicks' => 111, 'users' => 2),
    array('host' => 2, 'country' => 'es', 'year' => 2010,
        'month' => 2, 'clicks' => 2,   'users' => 4),
    array('host' => 3, 'country' => 'es', 'year' => 2010,
        'month' => 3, 'clicks' => 34,  'users' => 2),
    array('host' => 3, 'country' => 'es', 'year' => 2010,
        'month' => 4, 'clicks' => 1,   'users' => 1),
);
```

| **host** | **country** | **year** | **month** | **clicks** | **users** |
| --- | --- | --- | --- | --- | --- |
| 1 | fr | 2010 | 1 | 123 | 4 |
| 1 | fr | 2010 | 2 | 134 | 5 |
| 1 | fr | 2010 | 3 | 341 | 2 |
| 1 | es | 2010 | 1 | 113 | 4 |
| 1 | es | 2010 | 2 | 234 | 5 |
| 1 | es | 2010 | 3 | 421 | 2 |
| 1 | es | 2010 | 4 | 22 | 3 |
| 2 | es | 2010 | 1 | 111 | 2 |
| 2 | es | 2010 | 2 | 2 | 4 |
| 3 | es | 2010 | 3 | 34 | 2 |
| 3 | es | 2010 | 4 | 1 | 1 |

And I need the information grouped by host and show it in a table with: each row one host and each column the sum of clicks and users per month

The interface I have created is the following one:

```php
$data = Pivot::factory($recordset)
    ->pivotOn(array('host'))
    ->addColumn(array('year', 'month'), array('users', 'clicks',))
    ->fetch();
```

Let's explain the interface:

Loads the original recorset into the class:

```php
Pivot::factory($recordset)
```

I want to pivot the recordset on the field 'host'

```php
->pivotOn(array('host'))
```

The columns are grouped by year and month and each group will show the sum of users and clicks

```php
->addColumn(array('year', 'month'), array('users', 'clicks',))
```

And finally I make the transformation:

```php
->fetch();
```

Basically that's it. I also has made some other helpers to: Add totals per line Add total per pivoted colum Add total for all table

```php
->fullTotal()
->pivotTotal()
->lineTotal()
```

In this class I allow to pivot tables with one, two or three fields. And also I allow to use calculated columns. For example imagine you need to show users, clicks and the average of clicks per user. It's easy clicks/users but you must take care in the calculated column because you can't sum it. You must calculate the calculated column over the sum of clicks and user. Because of that I define calculated functions. This feature only works with PHP5.3 because it uses closures.

```php
$averageCbk = function($reg)
{
    return round($reg['clicks']/$reg['users'],2);
};
 
$data = Pivot::factory($recordset)
    ->pivotOn(array('host', 'country'))
    ->addColumn(array('year', 'month'), array('users', 'clicks', Pivot::callback('average', $averageCbk)))
    ->lineTotal()
    ->pivotTotal()
    ->fullTotal()
    ->typeMark()
    ->fetch();
```

And now different usage examples:

pivot on 'host'

```php
$data = Pivot::factory($recordset)
    ->pivotOn(array('host'))
    ->addColumn(array('year', 'month'), array('users', 'clicks',))
    ->fetch();
```

| **\_id** | **host** | **2010\_1\_users** | **2010\_1\_clicks** | **2010\_2\_users** | **2010\_2\_clicks** | **2010\_3\_users** | **2010\_3\_clicks** | **2010\_4\_users** | **2010\_4\_clicks** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 8 | 236 | 10 | 368 | 4 | 762 | 3 | 22 |
| 2 | 2 | 2 | 111 | 4 | 2 |  |  |  |  |
| 3 | 3 |  |  |  |  | 2 | 34 | 1 | 1 |

pivot on 'host' with totals

```php
$data = Pivot::factory($recordset)
    ->pivotOn(array('host'))
    ->addColumn(array('year', 'month'), array('users', 'clicks',))
    ->fullTotal()
    ->lineTotal()
    ->fetch();
```

| **\_id** | **host** | **2010\_1\_users** | **2010\_1\_clicks** | **2010\_2\_users** | **2010\_2\_clicks** | **2010\_3\_users** | **2010\_3\_clicks** | **2010\_4\_users** | **2010\_4\_clicks** | **TOT\_users** | **TOT\_clicks** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 8 | 236 | 10 | 368 | 4 | 762 | 3 | 22 | 25 | 1388 |
| 2 | 2 | 2 | 111 | 4 | 2 |  |  |  |  | 6 | 113 |
| 3 | 3 |  |  |  |  | 2 | 34 | 1 | 1 | 3 | 35 |
| 4 | TOT | 10 | 347 | 14 | 370 | 6 | 796 | 4 | 23 | 34 | 1536 |

pivot on 'host' and 'country'

```php
$data = Pivot::factory($recordset)
    ->pivotOn(array('host', 'country'))
    ->addColumn(array('year', 'month'), array('users', 'clicks',))
    ->fullTotal()
    ->pivotTotal()
    ->lineTotal()
    ->fetch();
```

| **\_id** | **host** | **country** | **2010\_1\_users** | **2010\_1\_clicks** | **2010\_2\_users** | **2010\_2\_clicks** | **2010\_3\_users** | **2010\_3\_clicks** | **2010\_4\_users** | **2010\_4\_clicks** | **TOT\_users** | **TOT\_clicks** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | fr | 4 | 123 | 5 | 134 | 2 | 341 |  |  | 11 | 598 |
| 2 | 1 | es | 4 | 113 | 5 | 234 | 2 | 421 | 3 | 22 | 14 | 790 |
| 3 | TOT(host) |  | 8 | 236 | 10 | 368 | 4 | 762 | 3 | 22 | 25 | 1388 |
| 4 | 2 | es | 2 | 111 | 4 | 2 |  |  |  |  | 6 | 113 |
| 5 | TOT(host) |  | 2 | 111 | 4 | 2 | 0 | 0 | 0 | 0 | 6 | 113 |
| 6 | 3 | es |  |  |  |  | 2 | 34 | 1 | 1 | 3 | 35 |
| 7 | TOT(host) |  | 0 | 0 | 0 | 0 | 2 | 34 | 1 | 1 | 3 | 35 |
| 8 | TOT |  | 10 | 347 | 14 | 370 | 6 | 796 | 4 | 23 | 34 | 1536 |

pivot on 'host' and 'country' with calculated columms

```php
$averageCbk = function($reg)
{
    return round($reg['clicks']/$reg['users'],2);
};
 
$data = Pivot::factory($recordset)
    ->pivotOn(array('host', 'country'))
    ->addColumn(array('year', 'month'), array('users', 'clicks', Pivot::callback('average', $averageCbk)))
    ->lineTotal()
    ->pivotTotal()
    ->fullTotal()
    ->typeMark()
    ->fetch();
```

| **\_id** | **type** | **host** | **country** | **2010\_1\_users** | **2010\_1\_clicks** | **2010\_1\_average** | **2010\_2\_users** | **2010\_2\_clicks** | **2010\_2\_average** | **2010\_3\_users** | **2010\_3\_clicks** | **2010\_3\_average** | **2010\_4\_users** | **2010\_4\_clicks** | **2010\_4\_average** | **TOT\_users** | **TOT\_clicks** | **TOT\_average** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 0 | 1 | fr | 4 | 123 | 30.75 | 5 | 134 | 26.8 | 2 | 341 | 170.5 |  |  | 0 | 11 | 598 | 54.36 |
| 2 | 0 | 1 | es | 4 | 113 | 28.25 | 5 | 234 | 46.8 | 2 | 421 | 210.5 | 3 | 22 | 7.33 | 14 | 790 | 56.43 |
| 3 | 1 | TOT(host) |  | 8 | 236 | 29.5 | 10 | 368 | 36.8 | 4 | 762 | 190.5 | 3 | 22 | 7.33 | 25 | 1388 | 264.13 |
| 4 | 0 | 2 | es | 2 | 111 | 55.5 | 4 | 2 | 0.5 |  |  | 0 |  |  | 0 | 6 | 113 | 18.83 |
| 5 | 1 | TOT(host) |  | 2 | 111 | 55.5 | 4 | 2 | 0.5 | 0 | 0 | 0 | 0 | 0 | 0 | 6 | 113 | 56 |
| 6 | 0 | 3 | es |  |  | 0 |  |  | 0 | 2 | 34 | 17 | 1 | 1 | 1 | 3 | 35 | 11.67 |
| 7 | 1 | TOT(host) |  | 0 | 0 | 0 | 0 | 0 | 0 | 2 | 34 | 17 | 1 | 1 | 1 | 3 | 35 | 18 |
| 8 | 3 | TOT |  | 10 | 347 | 34.7 | 14 | 370 | 26.43 | 6 | 796 | 132.67 | 4 | 23 | 5.75 | 34 | 1536 | 45.18 |

**UPDATED**

As Marcin said in a [comment](http://gonzalo123.wordpress.com/2010/01/24/pivot-tables-in-php/#comment-163) we can add the count of grouped columns. This functionality was not available on the first release of Pivot class. I think it can be useful so I've developed it Some examples:

pivot on 'host' and 'country' with group count

```php
$data = Pivot::factory($recordset)
    ->pivotOn(array('host', 'country'))
    ->addColumn(array('year', 'month'), array('users', 'clicks', Pivot::count('count')))
    ->fullTotal()
    ->pivotTotal()
    ->lineTotal()
    ->fetch();
```

| **\_id** | **host** | **country** | **2010\_1\_users** | **2010\_1\_clicks** | **2010\_1\_count** | **2010\_2\_users** | **2010\_2\_clicks** | **2010\_2\_count** | **2010\_3\_users** | **2010\_3\_clicks** | **2010\_3\_count** | **2010\_4\_users** | **2010\_4\_clicks** | **2010\_4\_count** | **TOT\_users** | **TOT\_clicks** | **TOT\_count** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | fr | 4 | 123 | 1 | 5 | 134 | 1 | 2 | 341 | 1 |  |  |  | 11 | 598 | 3 |
| 2 | 1 | es | 4 | 113 | 1 | 5 | 234 | 1 | 2 | 421 | 1 | 3 | 22 | 1 | 14 | 790 | 4 |
| 3 | TOT(host) |  | 8 | 236 | 2 | 10 | 368 | 2 | 4 | 762 | 2 | 3 | 22 | 1 | 25 | 1388 | 7 |
| 4 | 2 | es | 2 | 111 | 1 | 4 | 2 | 1 |  |  |  |  |  |  | 6 | 113 | 2 |
| 5 | TOT(host) |  | 2 | 111 | 1 | 4 | 2 | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 6 | 113 | 2 |
| 6 | 3 | es |  |  |  |  |  |  | 2 | 34 | 1 | 1 | 1 | 1 | 3 | 35 | 2 |
| 7 | TOT(host) |  | 0 | 0 | 0 | 0 | 0 | 0 | 2 | 34 | 1 | 1 | 1 | 1 | 3 | 35 | 2 |
| 8 | TOT |  | 10 | 347 | 3 | 14 | 370 | 3 | 6 | 796 | 3 | 4 | 23 | 2 | 34 | 1536 | 11 |

pivot on 'country' with group count

```php
$data = Pivot::factory($recordset)
    ->pivotOn(array('host'))
    ->addColumn(array('country'), array('year', Pivot::count('count')))
    ->lineTotal()
    ->fullTotal()
    ->fetch();
```

| **\_id** | **host** | **fr\_\_year** | **fr\_\_count** | **es\_\_year** | **es\_\_count** | **TOT\_year** | **TOT\_count** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 6030 | 3 | 8040 | 4 | 14070 | 7 |
| 2 | 2 |  |  | 4020 | 2 | 4020 | 2 |
| 3 | 3 |  |  | 4020 | 2 | 4020 | 2 |
| 4 | TOT | 6030 | 3 | 16080 | 8 | 22110 | 11 |

The source code of the class is available in [github](https://github.com/gonzalo123/gam-pivot)
