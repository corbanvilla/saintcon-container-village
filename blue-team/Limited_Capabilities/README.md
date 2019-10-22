# Limiting Container Capabilities

## What are Capabilities?

[`man 7 capabilties`](http://man7.org/linux/man-pages/man7/capabilities.7.html)
> "Starting with kernel 2.2, Linux divides the privileges traditionally associated with superuser into distinct units, known as capabilities, which can be independently enabled and disabled."

Docker uses capabilities to limit how containers can interact with the system. By default, many of the more dangerous capabilities are blocked. However, by fine-tuning your capabilities on a per-container basis, you can add another layer of security. 

**Note:** Refer to the [official Docker documentation](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities) for the most up-to-date list of capabilities that are allowed and denied by default.

Containers running as `root` have all of the default capabilities enabled, where running as a non-root user is the equivalent of adding the `--cap-drop ALL` option, meaning that they have none. This is one of the many reasons that running container as root is a bad idea. However, there are some cases where a container must run as root (it is not possible to add capabilities to non-root users). This is a situation where you would want to use `--cap-drop ALL` and then add required capabilities back with `--cap-add`. This way, your container will be running with the minimum privileges necessary. 

## Limiting Capabilities

Let's run an example. We've got a Python application that needs to run on port 80. 

Create a folder and name it drop_cap, then create the files `server.py` and `Dockerfile`.

```
mkdir drop_cap
cd drop_cap
```

Create `server.py`:
```
vim server.py
```
```
#!/usr/bin/env python3

import socket

HOST = '127.0.0.1'  # Standard loopback interface address (localhost)
PORT = 80        # Port to listen on (non-privileged ports are > 1023)

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.bind((HOST, PORT))
    print(f"listening on {HOST}:{PORT}")
    s.listen()
    conn, addr = s.accept()
    with conn:
        print('Connected by', addr)
        while True:
            data = conn.recv(1024)
            if not data:
                break
            conn.sendall(data)
```

Create your `Dockerfile`:
```
vim Dockerfile
```
```
FROM alpine:latest

WORKDIR /code

COPY server.py .

RUN apk add python3 && \
    rm -rf /var/cache/apk/

CMD ["/usr/bin/python3","/code/server.py"]
```

Then let's build the image and run it:

```
docker build . -t python-app

docker run -it --rm --name my_service python-app
```

You should get something that looks like the following:

```
listening on 127.0.0.1:80
```

`CTRL-C` to kill the container. 

Since we didn't specify another user in the Dockerfile, the container will run as `root`. And because we want to bind to port `80`, which is considered a privileged port, the process needs to run as `root`. Specifically, it needs the capability `NET_BIND_SERVICE`. 

However, the container will have all the other default capabilties as well, which could be leveraged as attack vectors down the road. 

For example, let's shell into the container and change the ownership properties of our file `server.py`:

```
docker run -it --rm --name my_service python-app /bin/sh
```
```
/code # chown nobody:nobody server.py
/code #
```

This runs fine because the capability `CHOWN` is another one of the default capabilities. 

If you attempt to create a `python` user like so:

```
FROM alpine:latest

WORKDIR /code

COPY server.py .

RUN apk add python3 && \
    rm -rf /var/cache/apk/ && \
    addgroup -S python && \
    adduser -S -G python python

USER python

CMD ["/usr/bin/python3","/code/server.py"]
```

You will receive a permission denied error:

```
Traceback (most recent call last):
  File "/code/server.py", line 9, in <module>
    s.bind((HOST, PORT))
PermissionError: [Errno 13] Permission denied
```

This is because when you run as a new user, it does not inherit ANY capabilities. 

Let's try running it as `root` but with some custom capabilities this time:

```
docker run -it --rm --name my_service --cap-drop ALL --cap-add NET_BIND_SERVICE python-app
```

**Note:** Notice the `--cap-drop ALL --cap-add NET_BIND_SERVICE` added to our runtime flags. 

You should get a message like this:

```
...
listening on 127.0.0.1:80
...
```

However, with these same runtime flags lets shell in and try running `chown`:

```
docker run -it --rm --name my_service --cap-drop ALL --cap-add NET_BIND_SERVICE python-app /bin/sh
```

```
/code # chown nobody:nobody server.py
chown: server.py: Operation not permitted
/code # whoami
root
```

The default capabilities were created to be general enough to work for most applications you would want to run without causing permission issues. However, leaving them alone leaves several security holes in your containers. For example, we know that there's never a time that our application will need to use `chown`, so leaving it enabled is just another possible attack vector. 