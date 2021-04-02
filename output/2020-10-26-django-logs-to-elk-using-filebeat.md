---
title: "Django logs to ELK using Filebeat"
date: "2020-10-26"
categories: 
  - "technology"
tags: 
  - "django"
  - "elk"
  - "python"
coverImage: "kibana.png"
---

I've written a post about how to send Django logs to ELK stack. You can read it [here](https://gonzalo123.com/2020/08/10/monitoring-django-applications-with-grafana-and-kibana-using-prometheus-and-elasticsearch/). In that post I've used logstash client with a sidecar docker container. Logstash client works but it needs too much resources. Nowadays it's better to use [Filebeat](https://www.elastic.co/beats/filebeat) as data shipper instead of Logstash client. Filebeat it's also a part of ELK stack. It's a golang binary much lightweight than logstash client.

The idea is almost the same than the other post. Here we'll also build a sidecar container with our django application logs mounted.

version: '3'
services:
  # Application
  api:
    image: elk:latest
    command: /bin/bash ./docker-entrypoint-wsgi.sh
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DEBUG: 'True'
    volumes:
      - logs\_volume:/src/logs
      - static\_volume:/src/staticfiles
  nginx:
    image: elk-nginx:latest
    build:
      context: .docker/nginx
      dockerfile: Dockerfile
    volumes:
      - static\_volume:/src/staticfiles
    ports:
      - 8000:8000
    depends\_on:
      - api
  filebeat:
    image: filebeat:latest
    build:
      context: .docker/filebeat
      dockerfile: Dockerfile
    volumes:
      - logs\_volume:/app/logs
    command: filebeat -c /etc/filebeat/filebeat.yml -e -d "\*" -strict.perms=false
    depends\_on:
      - api

  ...

With filebeat we can perform actions to prepare our logs to be ready to be stored within elasticsearch. But, at least here, it's much more easy to prepare the logs in the django application:

class CustomisedJSONFormatter(json\_log\_formatter.JSONFormatter):
    def json\_record(self, message: str, extra: dict, record: logging.LogRecord):
        context = extra
        django = {
            'app': settings.APP\_ID,
            'name': record.name,
            'filename': record.filename,
            'funcName': record.funcName,
            'msecs': record.msecs,
        }
        if record.exc\_info:
            django\['exc\_info'\] = self.formatException(record.exc\_info)

        return {
            'message': message,
            'timestamp': now(),
            'level': record.levelname,
            'context': context,
            'django': django
        }

And in settings.py we use our CustomisedJSONFormatter

LOGGING = {
    'version': 1,
    'disable\_existing\_loggers': False,
    'formatters': {
        'simple': {
            'format': '\[%(asctime)s\] %(levelname)s|%(name)s|%(message)s',
            'datefmt': '%Y-%m-%d %H:%M:%S',
        },
        "json": {
            '()': CustomisedJSONFormatter,
        },
    },
    'handlers': {
        'applogfile': {
            'level': 'DEBUG',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': Path(BASE\_DIR).resolve().joinpath('logs', 'app.log'),
            'maxBytes': 1024 \* 1024 \* 15,  # 15MB
            'backupCount': 10,
            'formatter': 'json',
        },
        'console': {
            'level': 'DEBUG',
            'class': 'logging.StreamHandler',
            'formatter': 'simple'
        }
    },
    'root': {
        'handlers': \['applogfile', 'console'\],
        'level': 'DEBUG',
    }
}

And that's all. Our Application logs centralized in ELK and ready to consume with Kibana

![](https://gonzalo123.files.wordpress.com/2020/09/kibana.png?w=1024)

Source code available [here](https://github.com/gonzalo123/django-logs-filebeat)
