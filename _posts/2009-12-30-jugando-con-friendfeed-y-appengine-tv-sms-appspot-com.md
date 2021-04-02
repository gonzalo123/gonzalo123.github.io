---
layout: post
permalink: /:year/:month/:day/:title/
title: "Jugando con FriendFeed y Appengine: tv-sms.appspot.com"
date: "2009-12-30"
categories: 
  - "appengine"
  - "web-development"
  - "webdev"
---

# INTRODUCCIÓN

Estas navidades me he propuesto publicar un aplicación con google app engine. La idea de la aplicación es simple. Hoy en día es muy común disponer de un portátil o netbook y estar en la sala viendo la tele con nuestro portátil. Los canales de televisión suelen permitir que la gente mande sms a unos números de teléfono y estos mensajes aparezcan en la parte inferior de nuestras pantallas. Personalmente no lo he usado nunca pero parece ser que el servicio tiene mucho éxito. Estos mensajes tienen un coste bastante elevado y son una fuente de ingresos para los canales y para las operadoras. La idea de la aplicación es hacer esto por Internet agarrándonos los sms aprovechando que tenemos un portátil en la sala junto a nosotros.

Los requisitos que me he impuesto son los siguientes:

- No dispongo de tiempo ilimitado. Tengo que publicarla en un par de días.
- Usar la [API](http://friendfeed.com/api/documentation) de FriendFeed.
- La aplicación tiene que limitar en lo posible el uso de recursos en el servidor.
- Publicar el [código](http://code.google.com/p/gam-tvsms/) en google code.
- Subir la aplicación a appengine: [http://tv-sms.appspot.com/](http://tv-sms.appspot.com/)
- Escribir una entrada en mi [blog](http://wp.me/pqmgV-2U) explicando como esta hecha.

Pues bien ya estoy en el ultimo punto así que paso a explicar como funciona.

# FUNCIONAMIENTO DE LA APLICACIÓN

## Creación de los grupos

La idea es la siguiente me creo con mi usuario un grupo público en FriendFeed por cada canal de televisión y después me creo un archivo javascript con la configuración de los canales:

```javascript
var chanels = {
    'tvsmstve1' : 'TVE 1',
    'tvsmstve2' : 'TVE 2',
    'tvsms-antena3' : 'Antena 3',
    'tvsmscuatro' : 'Cuatro',
    'tvsms-tele5' : 'Tele 5',
    'tvsmslasexta' : 'La Sexta'
}
```

Entiendo que esta información la debería guardar en algún tipo de Base de datos pero por ahora se queda así. Al arrancar la aplicación leo esto y lo muestro en pantalla

```javascript
function populateChanels() {
    $.each(chanels, function(i, chanel) {
        $('#chanels').append("
    <li class="status"><a class="chanelsLink" onclick="start(\&quot;&quot; + i + &quot;\&quot;)" href="#">" + chanel + "</a></li>
");
    });
}
```

## Listado de las últimas modificaciones de los grupos

Friendfeed tiene una API muy sencilla de usar usando REST y JSONP. Para obtener una la lista de actualizaciones de un canal simplemente tenemos que hacer una petición GET a la siguiente url:

> http://friendfeed-api.com/v2/feed/grupoid

Por lo tanto y teniendo en cuenta que voy a utilizar jQuery para mi aplicación creo la siguiente función:

```javascript
var users = {};
var frmInput = "
<div class="share"><form action="?" method="post"><input id="body" style="width: 300px;" name="body" type="text" /><input id="submit" type="submit" value="Post" /></form></div>
";
 
function _getLi(body, user, date) {
    var fDade = formatFriendFeedDate(date);
    return "
    <li class="status"><span class="thumb vcard author"><img height='50' width='50' src='http://friendfeed-api.com/v2/picture/" + user + "?size=medium' alt='" + user + "'/></span><span class="status-body">" + user + " : " + body + "<span class="meta entry-meta">" + fDade + "</span></span></li>
";
}
 
function list(id) {
    var txt = '';
    $.getJSON("http://friendfeed-api.com/v2/feed/" + id + "?callback=?",
        function (data) {
            $('#ff').html('');
            $.each(data.entries, function(i, entry) {
                users[entry.from.id] = entry.from.id;
                txt+= _getLi(entry.body, entry.from.id, entry.date);
            });
            $("#ff").html(txt);
            if (logged) {
                $('#inputForm').html(frmInput);
            }
            $('#txtCanal').html(chanels[id]);
 
            cometClient(id);
        });
}
```

En esta función lo que estoy haciendo es obtener la lista en formato JSON y añadirla a un ol. Además de la lista le añado un formulario para que el usuario pueda introducir comentarios y arranco el cliente comet pero esto lo explicaré más adelante.

## Actualizaciones en tiempo real

Lo que mas me gusta de Friendfeed y lo que me ha hecho decantarme por el y no por Twitter es que Friendfeed me permite lo que llama [Real-Time updates](http://friendfeed.com/api/documentation#realtime). Esta caracteristica, que hasta donde yo se Twitter no nos la da nos permite hacer de forma muy sencilla clientes Comet. Aquí va el mio:

```javascript
function cometClient(id, cursor) {
    var url = "http://friendfeed-api.com/v2/updates/feed/" + id + "?callback=?&timeout=5";
    if (cursor) {
        url += "&cursor==" + cursor;
    }
    $.getJSON(url, function (data) {
        $.each(data.entries, function(i, entry) {
            $('#ff').prepend(_getLi(entry.body, entry.from.id, entry.date));
        });
        t = setTimeout(function() {cometClient(id, data.realtime.cursor)}, 2000);
    });
}
```

Pues nada gracias a esto tengo actualizaciones a tiempo real de mi aplicación y sin usar ningún recurso de servidor.

## Nuevo comentario

Para crear un nuevo comentario la API de FriendFeed no me deja hacerlo directamente desde javascript. Esto es una pequeña faena ya que si no tuviera esta restricción toda la aplicación correría sobre javascript y no necesitaría nada de código en servidor. Pero al no ser posible me tengo que autentificar con OAuth desde google appengine. Para hacer esto me baso en la aplicación de [ejemplo](http://code.google.com/p/friendfeed-api-example/) que nos proporciona FriendFeed para jugar con su API.

El citado ejemplo hace todo lo que necesito y lo hace todo en el servidor. Ya que yo solo voy a usar la parte que FriendFeed denomina creating an entry, me limito a leer el código y eliminar la parte que no necesito. Como una de mis premisas es que no dispongo de tiempo ilimitado para esta aplicación no me voy a dedicar a entender cada linea de código. Me dedico a entender mas o menos como funciona.

En mi aplicación la opción de nuevo comentario va a ser llamada desde javascript y espero un JSON por lo que a parte de eliminar las partes del código que no me interesan, hago unas pequeñas modificaciones para que la respuesta sea JSON

```python
class EntryHandler(webapp.RequestHandler):
    @authenticated
    def post(self):
        try:
            entry = self.friendfeed.post_entry(
                body=self.request.get("body"),
                to=self.request.get("to"))
            out = {'error' : 0}
        except:
            out = {"redirect" : '/oauth/authorize'}
        self.response.headers['Content-Type'] = 'application/json'
        jsonData = out
        self.response.out.write(simplejson.dumps(jsonData))
        return
...
 
application = webapp.WSGIApplication([
    (r"/oauth/callback", OAuthCallbackHandler),
    (r"/oauth/authorize", OAuthAuthorizeHandler),
    (r"/oauth/check", OAuthCheck),
    (r"/a/entry", EntryHandler),
])
```

El código es bastante mejorable (no hay gestión de errores y no se hace nada con la variable entry) pero hace lo que quiero: publica una entrada en FriendFeed.

**Autentificando**

Para publicar entradas en FrinedFeed necesito estar autentificado por lo cual quiero que la aplicación solo muestre el formulario de entrada de datos si el usuario esta autentificado. Para hacer esto hago una llamada asíncrona al servidor al cargar la aplicación que me mire si el usuario esta o no autentificado

```javascript
function checkFfLogin() {
    $.post("/oauth/check", {}, function (data) {
        if (data.redirect) {
            logged = false;
        } else {
            if (data.ok && data.ok==1) {
                logged = true;
            }
        }
 
        if (logged == true) {
            $('#login').html('');
        } else {
            $('#login').html("<a href="&quot; + data.redirect + &quot;"><img border='0' src='/sign-in-with-friendfeed.png' alt='Sign in with FriendFeed'></a>");
        }
        populateChanels();
    }, "json");
}
```

Esto se complementa con la parte del servidor.

```python
class OAuthCheck(webapp.RequestHandler):
    @authenticated
    def post(self):
        cookie_val = parse_cookie(self.request.cookies.get("FF_API_AUTH"))
        try:
            key, secret, username = cookie_val.split("|")
            self.response.headers['Content-Type'] = 'application/json'
            jsonData = {"ok" : 1}
            self.response.out.write(simplejson.dumps(jsonData))
        except:
            self.response.headers['Content-Type'] = 'application/json'
            jsonData = {"redirect" : '/oauth/authorize'}
            self.response.out.write(simplejson.dumps(jsonData))
        return
```

**Cambio de canal**

Ya solo queda que el usuario pueda cambiar de canal.

```javascript
var selectedChanel;
function start(group) {
    $('#body').val('');
    selectedChanel = group;
    if (t) {
        clearTimeout(t);
    }
    $('#ff').html('<img src="/ajax-loader.gif" alt="loader"/>... cargando ' + chanels[group]);
 
    list(group);
}
```

# FIN

Pues bien la aplicación ya esta operativa. Entiendo que tiene bastantes lagunas pero funciona y he cumplido las premisas que me había propuesto: Un par de tardes para hacerla y otra más para escribir la entrada en el blog y subirla al appengine.

Enlace a la aplicación: [http://tv-sms.appspot.com/](http://tv-sms.appspot.com/) Código fuente: [http://code.google.com/p/gam-tvsms/](http://code.google.com/p/gam-tvsms/)
