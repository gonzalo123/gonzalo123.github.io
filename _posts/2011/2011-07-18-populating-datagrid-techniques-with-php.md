---
layout: post
title: "Populating datagrid techniques with PHP"
date: "2011-07-18"
tags: 
  - "jquery"
  - "php"
  - "technology"
  - "tips"
  - "web-development"
  - "webdev"
---

Today I want to speak about populating datagrid techniques with PHP. At least in my daily work datagrids and tabular data are very common, because of that I want to show two different techniques when populating datagrids with data from our database. Maybe it's obvious, but I want to show the differences. Let's start.

Imagine we need to fetch data from our database and show it in a datagrid. Let's do the traditional way. I haven't use any framework for this example. Just old school spaghetti code.

```php
$dbh = new PDO('pgsql:dbname=mydb;host=localhost', 'gonzalo', 'password');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$stmt = $dbh->prepare('SELECT * FROM test.tbl1 limit 10');
$stmt->setFetchMode(PDO::FETCH_ASSOC);
$stmt->execute();
 
$data = $stmt->fetchAll();
 
$table = "";
$table.= "<table>";
foreach ($data as $ow) {
    $table.= "<tr>";
        foreach ($ow as $item) {
            $table.= "<td>{$item}</td>";
        }
    $table.= "</tr>";
}
$table.= "</table>";
?>
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
        <title>inline grid</title>
    </head>
    <h1>inline grid</h1>
    <body>
        <?php echo $table; ?>
        <script type="text/javascript" src=" https://ajax.googleapis.com/ajax/libs/jquery/1.6.2/jquery.min.js"></script>
    </body>
</html>
```

And that's all. The code works, and we've got or ugly datagrid with the data from our DB. Where's the problem? If our "SELECT" statement is fast enougth and our connection to the database is good too, the page load will be good, indeed. But what happens if or query is slow (or we even have more than one)? The whole page load will be penalized due to our slow query. The user won't see anything until our server finish with all the work. That means bad user experience. The alternative is load the page first (without populated datagrid, of course) and when it's ready, we load with ajax the data from the server (JSON) and we populate the datagrid with javaScript.

Page without the populated datagrid: 

```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
        <title>grid ajax</title>
    </head>
    <h1>grid ajax</h1>
    <body>
        <table id='grid'></table>
 
        <script type="text/javascript" src=" https://ajax.googleapis.com/ajax/libs/jquery/1.6.2/jquery.min.js"></script>
        <script type="text/javascript">
        $(function(){
            $.getJSON("json.php", function(json){
                        for (var i=0;i<json.length;i++) {
                            $('#grid').append("<tr><td>" + json[i].id + "</td><td>" + json[i].field1 + "</td></tr>")
                        }
                    });
        });
        </script>
    </body>
</html>
```

JSON data fron the server: 

```php
$dbh = new PDO('pgsql:dbname=mydb;host=localhost', 'gonzalo', 'password');
$dbh->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
 
$stmt = $dbh->prepare('SELECT * FROM test.tbl1 limit 10');
$stmt->setFetchMode(PDO::FETCH_ASSOC);
$stmt->execute();
 
$data = $stmt->fetchAll();
 
header('Cache-Control: no-cache, must-revalidate');
header('Expires: Mon, 26 Jul 1997 05:00:00 GMT');
header('Content-type: application/json');
 
echo json_encode($data);
```

The outcome of this second technique is the same than the first one, but now user will see the page faster than the first technique and data will load later. Probably the total time to finish all the work is better in the classical approach, but the UX will be better with the second one. Here you can see the times taken from Chorme's network window:

[](/assets/images/inline.png "inline")

[](/assets/images/ajax3.png "ajax3")

Even though the total time is better in the inline grid example: _156ms vs 248ms, 1 http request vs 3 HTTP request_. The user will see the page (without data) faster with the grid data example.

What's better. As always it depends on our needs. We need to balance and choose the one that fits with your requirements.

Probably the JavaScript code that I use to populate the datagrid in the second technique could be written in a more efficient way (there's also plugins to do it). I only want to show the indirect way to create HTML to improve the user experience of our applications.
