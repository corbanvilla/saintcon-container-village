# gVisor

## What is gVisor?

gVisor is a project that Google released which implements a user-space kernel for containers to make calls to, rather than the host kernel. 

>gVisor is a user-space kernel, written in Go, that implements a substantial portion of the Linux system surface. It includes an Open Container Initiative (OCI) runtime called `runsc` that provides an isolation boundary between the application and the host kernel. The `runsc` runtime integrates with Docker and Kubernetes, making it simple to run sandboxed containers.

>gVisor intercepts application system calls and acts as the guest kernel... gVisor may be thought of as either a merged guest kernel and VMM, or as seccomp on steroids. This architecture allows it to provide a flexible resource footprint (i.e. one based on threads and memory mappings, not fixed guest physical resources) while also lowering the fixed costs of virtualization. However, this comes at the price of reduced application compatibility and higher per-system call overhead.

Google has been using gVisor in production for a few years now, so it is considered to be relatively robust. It works by replacing the default container runtime `runc`, and will work as a drop-in replacement for most scenarios. 

## Installing gVisor

Use the following script to install the latest version of gVisor:

```
vim install_gvisor.sh
```

```bash
#!/bin/bash
set -e

# Download binary and hash
wget https://storage.googleapis.com/gvisor/releases/nightly/latest/runsc
wget https://storage.googleapis.com/gvisor/releases/nightly/latest/runsc.sha512

# Confirm hashes match, fail if they don't
sha512sum -c runsc.sha512
rm runsc.sha512

# The runsc binary executes as user: nobody
sudo mv runsc /usr/local/bin
sudo chown root:root /usr/local/bin/runsc
sudo chmod 0755 /usr/local/bin/runsc

echo "gVisor successfully installed!"
```

Then go ahead and run it:

```bash
chmod +x install_gvisor.sh

./install_gvisor.sh
```

Now we need to point Docker to the `runsc` binary that we just installed, and set `runsc` as the default. Do this in your `/etc/docker/daemon.json` like so:

```
sudo vim /etc/docker/daemon.json
```

```json
{
    "default-runtime": "runsc",
    "runtimes": {
        "runsc": {
            "path": "/usr/local/bin/runsc"
        }
    }
}
```

Go ahead and restart your Docker daemon to pick up the new changes. 

**Note:** Any running containers will also need to be destroyed and re-created. 

```
sudo systemctl restart docker
```

And confirm that it's up and running again:

```
sudo systemctl status docker
```

Verify that it's up and running successfully by starting a new container and checking `dmesg`:

```
docker run -it --rm ubuntu dmesg
```

You should see something like the following:

```
root@localhost:~# docker run -it ubuntu dmesg
[    0.000000] Starting gVisor...
```

Congratulations! You've now configured gVisor as the default docker runtime. 