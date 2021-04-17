---
layout: post
title: "Real-life example of Closure usage with PHP5.3"
date: "2010-12-06"
tags: 
  - "php"
categories: 
  - "php"
---

One of the new improvements in new PHP5.3 version are the **Closures**. Here you are a real-life example where closures are really useful for me. Imagine the following scenario. A very common scenario at least in my daily work. We have a recordset from a SQL:

```php
$recordset = array(
    array('code' => 1, 'quantity' => 20, 'amount' => 2),
    array('code' => 2, 'quantity' => 30, 'amount' => 3),
    array('code' => 3, 'quantity' => 10, 'amount' => 1),
    array('code' => 4, 'quantity' => 20, 'amount' => 5),
)
```

Imagine we want to show the recordset into a datagrid but we also want to calculate the price (quality / amount) of each row and add a total. We can perform the operation within the SQL but I prefer to keep the SQLs as simple as I can and let the logic to PHP.

We can use [array\_walk](http://es.php.net/manual/en/function.array-walk.php) to calculate the price but with PHP<5.3 we needed to pass the callback function as a string. A dirty trick. Maybe a simple _foerach_ is a better solution, isn’t it?. Now with PHP5.3 we can create anonymous functions (aka _Closures_) in the finest _JavaScript_ style. They are really cool. We even can use the ‘_use_’ statement to use variables outside Closuse’s scope. This attribute allows us in our example to calculate our totals.

Maybe is difficult to explain but the code is very clear:

```php
$total = array('quantity' => 0, 'amount' => 0, 'price' => null);
array_walk($recordset, function (&$value) use(&$total){
    $value['price'] = ($value['amount'] == 0) ? null : $value['quantity'] / $value['amount'];
    $total['quantity'] += $value['quantity'];
    $total['amount'] += $value['amount'];
});
$total['price'] = ($total['amount'] == 0) ? null : $total['quantity'] / $total['amount'];
```

And now our $recordset array will have an extra item called register and we have created a new row for our recordset with total values.

**recordset**:

```
Array(
 [0]=>Array(=>1 [quantity]=>20 [amount]=>2 [price]=>10)
 [1]=>Array(=>2 [quantity]=>30 [amount]=>3 [price]=>10)
 [2]=>Array(=>3 [quantity]=>10 [amount]=>1 [price]=>10)
 [3]=>Array(=>4 [quantity]=>20 [amount]=>5 [price]=>4)
)
```

**total**:

```
Array(
 [quantity] => 80 [amount]=>11 [price] => 7.2727272727273
)
```
