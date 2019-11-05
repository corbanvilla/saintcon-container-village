# Log Aggregation

## What is Log Aggregation?
A core component of security is configruing log aggregation. Log aggregation is the process of gathering log files from many sources and centralizing them. Good log aggregation provides incredible insight into your systems. It allows you to quickly diagnose issues, alert you when applications are sending errors, and gives you an audit trail to review comprimised systems. 

Common log aggregation systems include:
- [Graylog](https://www.graylog.org/) - An open-source log aggregation tool backed with Elasticsearch, written in Java
- [Splunk](https://www.splunk.com/) - An excellent and expensive log aggregation tool with some very powerful features
- [ELK Stack](https://www.elastic.co/what-is/elk-stack) - Short for Elasticsearch, Logstash, and Kibana, an open-source and complex yet powerful tool for search and analytics

**Note:** Though this guide walks you through installing Graylog, Splunk and ELK are both very robust options as well. If you'd like to configure those instead, come and show us and we will you with equal points. 

## Graylog Install

In order to install Graylog, I have a precompiled `docker-compose.yml` file to bootstrap Graylog. When using Graylog in production, you will want to review [the Graylog install docs](https://docs.graylog.org/en/3.1/pages/installation.html) along with [the architectural design docs](https://docs.graylog.org/en/3.1/pages/architecture.html).

In short--the Graylog container is running a Java frontend that you connect to, along with the endpoints that receives and processes logs. MongoDB is storing configuration data, and Elasticsearch is what stores your logs and is the engine behind searching and analyzing. 

**Note:** I recognize simply copy-pasting a compose-file does not give you the same understanding of a system that building it from the ground-up does. However, because of the varying skill-level and limited time we have, we're using a `docker-compose` file. If you would like to build a Graylog-server from the ground up (without Docker and on your own server so that others can use these) extra points will be awarded. When you have your server up and running, come and show us and we will award you with extra points. 

### Preparing the System
You need to up your systems nmapfs limit to allow Elasticsearch to run properly or you will run into out-of-memory errors. (As shown [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html).) Use the command:

```bash
sysctl -w vm.max_map_count=262144 
```

Then edit the file `/etc/sysctl.conf` and add `vm.max_map_count=262144` to make the change permanent:

```bash
sudo vim /etc/sysctl.conf

...
vm.max_map_count=262144
...
```

### Spinning up Graylog
Docker and docker-compose are already installed, so simply create a folder, copy the compose file and start Graylog:

```bash
mkdir graylog-server
cd graylog-server

vim docker-compose.yml
```

And paste the following:

```
version: '3'

volumes:
  elastic-search:
  graylog:
  graylog-plugins:
  mongo:

networks:
  graylog-net:

services:
  mongodb:
    image: mongo:3
    volumes:
      - mongo:/data/db
#    logging:
#      driver: "local"
#      options:
#        max-size: "500kB"
#        max-file: "3"
    networks:
      graylog-net:
        aliases:
          - mongo

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.2
    environment:
      - http.host=0.0.0.0
      - transport.host=localhost
      - network.host=0.0.0.0
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    # Remove maximum open file limit
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elastic-search:/usr/share/elasticsearch/data
#    logging:
#      driver: "local"
#      options:
#        max-size: "500kB"
#        max-file: "3"
    networks:
      graylog-net:
        aliases:
          - elasticsearch

  graylog:
    container_name: graylog
    image: graylog/graylog:3.1
    environment:
      - GRAYLOG_PASSWORD_SECRET=secure_with_graylog
      - GRAYLOG_ROOT_PASSWORD_SHA2=a64ece40da45cd7c48ac5a60db317df2026d6b5012c81debad9c92ccae89d1e5
      - GRAYLOG_HTTP_EXTERNAL_URI=http://192.168.105.238:9000/
      - GRAYLOG_ELASTICSEARCH_SHARDS=1
      - GRAYLOG_ELASTICSEARCH_REPLICAS=0
      - GRAYLOG_ALLOW_LEADING_WILDCARD_SEARCHES=true
      #- GRAYLOG_ROOT_TIMEZONE=MST7MDT # Default is UTC
    depends_on:
      - mongodb
      - elasticsearch
    ports:
      # Graylog web interface and REST API
      - 9000:9000
      # GELF UDP
      - 12201:12201/udp
    volumes:
      - graylog:/usr/share/graylog/data/journal
      - graylog-plugins:/usr/share/graylog/plugin/
#    logging:
#      driver: "local"
#      options:
#        max-size: "500kB"
#        max-file: "3"
    networks:
      graylog-net:
        aliases:
          - graylog
```

Then start up the server itself by typing:

```bash
docker-compose up -d && docker-compose logs -f
```

**Note:** By using `docker-compose up -d && docker-compose logs -f` it starts the service in the background, then starts following the logs, rather then `docker-compose up` which starts in the foreground. If started in the foreground, CTL-C will stop the service, but if you're only following the logs, the service will continue running, and it will only stop following the logs. 

The first time Graylog starts up it may take upwards of 5 minutes. Please be patient. You will also see some erorrs as Elasticsearch is coming online. This is to be expected. 

Once it is online, connect to it with the URL: `https://<your-server-ip>:9000`. The site is also slow while it initializes. Be patient. 

Log in with the username `admin`and the password `secure_with_graylog`

Congratulations! Graylog is now setup and ready to be configured!

### Configuring Graylog

First off, we need to enable a way for Graylog to listen for log messages.

Navigate to `System > Inputs`. Click the dropdown `Select input`and select `GELF UDP` then click `Launch new input`.

Name the input `Docker GELF`, then click `Save`. 

**NOTE:** GELF, or Graylog Extended Log Format is a logging format built by the developers over at Graylog. GELF has native support in the docker engine, and is very simple to configure and get running. However, you will notice a few shortcomings using the GELF driver. The first being that when it sends logs to Graylog, no local version is stored. This can make troubleshooting containers on a node slightly more difficult, as you would normally run `docker logs <container>`, you now have to reference Graylog. The other issue being multiline support. [Docker doesn't support multiline messages](https://github.com/moby/moby/issues/22920), and as a result you will find that every line is sent as an independent log with it's own entry and timestamp. This can be especially bothersome when looking through traceback errors. If you need multiline support, consider something like Filebeat that can parse logs then send them to Graylog first. Extra points awarded for configuring Filebeat to send container logs to Graylog. 

Notice in our `docker-compose.yml` file under ports, it includes:

```
      # GELF UDP
      - 12201:12201/udp
```

That means that it's already listening on port `12201` on the host and forwarding it to the container. 

### Configuring Clients

Now that Graylog is ready to accept connections, let's configure clients to send their Docker messages to Graylog. Disconnect from your Graylog server and connect over to a client. 

We'll use the file `/etc/docker/daemon.json` to make our configuration changes. It will be an empty file when you first open it because the Docker daemon is currently running the default configuration. Any changes you add to this file will overwrite defaults when the daemon is restarted. 

```bash
sudo nano /etc/docker/daemon.json

... 

{
  "log-driver": "gelf",
  "log-opts":  {
    "gelf-address": "udp://<your-graylog-server>:12201"
  }
}
```

**Note:** Further information on the GELF docker driver can be found [here](https://docs.docker.com/config/containers/logging/gelf/).

At this point you need to restart the Docker daemon for your changes to take effect:

```
sudo systemctl restart docker
```

Now it's time to start a new container and test out our logging. 
Let's start a simple NGINX container to test out:

```bash
docker run -it --rm --name nginx -p 80:80 nginx:latest
```

Once the container comes up, try connecting to the graylog-client IP in your webbrowser. You should see the defualt NGINX homepage. 

This event should generate 3 unique logs. Go to Graylog, then click `Streams > All messages`. You should see several log messages there now. Everything printed to `stdout` will be sent straight to Graylog. 

Congratulations! You've configured Graylog with Docker!