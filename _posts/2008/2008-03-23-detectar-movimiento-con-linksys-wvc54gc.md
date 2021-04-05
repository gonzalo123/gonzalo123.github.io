---
layout: post
permalink: /:year/:month/:day/:title/
title: "Detectar movimiento con Linksys WVC54GC"
date: "2008-03-23"
tags: 
  - "linux"
---

Tengo una cámara Linksys WVC54GC y estoy jugando con la detección del movimiento. Para esto he encontrado un script en bash que tocado un poquito aqui y allá, que hace mas o menos lo que necesito.

```shell
#!/bin/sh
 
FRAMERATE=2
SENSITIVITY=30
PNMPSNR=/usr/bin/pnmpsnr
cd /mnt/data/.cam/
while true
do
sleep $FRAMERATE
mplayer http://webcam/img/video.asf -really-quiet -frames 1 -vo jpeg
djpeg 00000001.jpg > current.ppm
date>Ycolour
$PNMPSNR current.ppm last.ppm>>Ycolour 2>
Y=`awk '/Y  color/ {print int($5)}' Ycolour`
if [ $Y -lt $SENSITIVITY ]
then
 
if [ -d ./`date +%Y%m%d` ]
then
cp 00000001.jpg ./`date +%Y%m%d`/`date +%y%m%d%H%M%S.jpg`
else
mkdir  ./`date +%Y%m%d`
cp 00000001.jpg ./`date +%Y%m%d`/`date +%y%m%d%H%M%S.jpg`
fi
 
fi
mv current.ppm last.ppm
```

Esto unido a un script en el init.d

```shell
donegonzalo@gnzl:/etc/init
gonzalo@gnzl:/etc/init.d$ cat webcamd
#!/bin/sh
### BEGIN INIT INFO
# Provides:          webcamd
# Required-Start:    networking
# Required-Stop:     networking
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start the webcamd web server.
### END INIT INFO
 
PATH=/sbin:/bin:/usr/sbin:/usr/bin
NAME=webcamd
DESC="webcam snapshot motion"
PIDFILE=/var/run/$NAME.pid
SCRIPTNAME=/etc/init.d/$NAME
ENV="env -i LANG=C PATH=/usr/local/bin:/usr/bin:/bin"
SSD="/sbin/start-stop-daemon"
 
FRAMERATE=2
SENSITIVITY=30
PNMPSNR=/usr/bin/pnmpsnr
. /lib/lsb/init-functions
cd /mnt/data/.cam/
case "$1" in
start)
      echo "Starting $DESC" $NAME
      su -l gonzalo -c "sh /mnt/data/.cam/startmotiondetection.sh &"
  ;;
*)
 
      exit 1
      ;;
esac
 
exit 0
```

Hace más o menos lo que quiero.

Si ya se que es muy mejorable ya que el tiempo que le he dedicado es muy poco (modo escusas activado). Si ya se que hay programas como Motion y Zoneminder que hacen esto mas bonito, pero bueno algún dia los miraré.

Mi principal problema es que la única forma que tengo para obtener una imagen fija de la camara es con el comando:

> mplayer http://webcam/img/video.asf -really-quiet -frames 1 -vo jpeg

ya que la cámara no me da la posibilidad de obterner la imagen directamente (o al menos no se como hacerlo). Mi problema es que esto no lo quiero ejecutar en un PC, como esta ahora, si no en un NSLU2, que es un aparatito con arquitectura ARM. Pues bien no consigo compilar el mplayer para ARM y encima creo que aunque lo consiga hacer no me va a funcionar ya que los codecs necesarios para ver un stream asf en Linux solo estan en formato binario para x86.

Me gustaría saber como se hace lo mismo que hago con mplayer con vlc (que si lo tengo correctamente instalado en el NSLU2). Otra opción es seguir las instrucciones que veo [aqui](http://www.infohit.net/blog/post/motion-capture-using-the-wvc54gc-with-linux.html) para cer funcionar la camarita con Motion pero no consigo hacer el reetreaming con ffmpeg, ya que al hacer:

> ffmpeg -an -i http://yourwebcam.up/img/video.asf http://localhost:8090/feed1.ffm

me dice que no puede abrir el archivo asf y no tengo ni idea por que.

En fin seguire peleando,
