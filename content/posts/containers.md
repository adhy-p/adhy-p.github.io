+++ 
date = 2022-12-09T14:22:26+08:00
title = "Exploring Containers"
description = ""
slug = ""
authors = []
tags = ["writeup", "linux"]
categories = []
externalLink = ""
series = []
+++



# Introduction to Containers

Containerisation is the process of packaging an application and the necessary resources (such as libraries and packages) required into one package.

The motivation behind containers is due to the myriad dependencies found in modern applications. These dependencies can:
- be difficult to install
- create difficulty for devs to diagnose and replicate faults
- conflict with each other (e.g. multiple versions of python)

Containerisation platforms make use of the `namespace` feature of the kernel, which is a feature used so that processes can access resources of the operating system without being able to interact with other processes.

Every process running on Linux will be assigned with two things: `namespace` and `PID`. Processes can only “see” other processes that are in the same namespace.

Actually, the concept of containerisation is not so new. Here's a very brief history of containers:

- 1979 - chroot system call
- 2000 - FreeBSD Jails
- 2001 - Linux Vserver
- 2004 - Solaris Zones
- 2008 - LXC
- 2013 - Docker

In 2013, Docker came out and become very popular because it makes containerisation very easy and accessible to developers without need to know the underlying kernel technologies such as namespaces and capabilities. Docker make use of existing Linux facilities to isolate each process from the underlying host and from each other.

These are the underlying concepts/technologies used to perform containterisation:
- Process
- Namespaces
    - Mount namespace: the process can only see certain directories
    - Network namespace: the process can only see certain network adapters
- Capabilities: break down root rights
    - In the old days: you are either root (all access), or not root (limited access)
    - Using capabilities, we allow a little bit of what root has without giving everything
    - In container context: we give every containers some capabilities
- Cgroups: restrict how much resources a process can use
    - By default, docker does not set the cgroup
    - Standard fork bomb will use all the process handles on the machine and DOS the machine
- AppArmor/SELinux: mandatory access control system
    - Restrict what a process can do. For instance, a web container can't create/delete files other than the ones in `/var/www`.
- Seccomp: block individual linux kernel syscalls
    - Docker blocks a set of “dangerous” linux kernel syscalls by default
    - Kubernetes does not implement this by default

Using these technologies, we can have a 'virtual environment' for our programs to run.

# Seeing Containers in Action
Now, we know that docker uses multiple Linux functionalities to achieve isolation. However, we can still 'break' the isolation barrier if we know what is going on.

Suppose we run a nginx webserver using a container:
```bash
# run a new docker container, with name 'webserver' with 'nginx' image
adhy$ docker run -d --name webserver nginx

# verify the docker process is running
adhy$ docker ps
CONTAINER ID    IMAGE    COMMAND    CREATED    STATUS    PORTS    NAMES
XXXXxxxxxxxx    nginx    xxxxxxx    xxxxxxx    Up        80/tcp   webserver  

```

The host can actually see that nginx processes are being spawned. This can be done by typing `ps` on the host machine.

```bash
adhy$ ps aux | grep -i nginx
root     ROOT_PID ROOT_PID ... nginx: master process nginx -g daemon off;
systemd+ PID      PID      ... nginx: worker process
systemd+ PID      PID      ... nginx: worker process
systemd+ PID      PID      ... nginx: worker process
```

Similarly, we can also explore the files inside a container. Suppose we create a new file on the root of the container by typing this command:
```
# docker exec will execute an command inside the container
# create a new file called 'my_new_file' in the root of webserver's file system
adhy$ docker exec webserver touch /my_new_file
```
We can access this file by accessing the `proc` filesystem. We simply need to find the PID of the docker process and we can go to the directory directly.
```
adhy$ sudo ls /proc/ROOT_PID/root
bin boot dev ... **my_new_file** ...
```

# Exploring Container Image
Usually, we run container from a container image. In essence, **A container image** are just **tarballs** with **json** metadata

We can create a tarball out of an image:
```bash
# create a tarball out of the docker nginx image
adhy$ docker save nginx:latest -o nginx-save.tar
```

Let's extract the tarball and explore the files:
```
adhy$ tar -xvf nginx-save.tar --one-top-level
adhy$ cd nginx-save
adhy$ ls
<sha256hash> # directories, one layer of dockerfile
<sha256hash> # directories
...
<sha256hash>.json
manifest.json
repositories

# using tree, we can see that each layer has another json file, an a layer.tar
```

We can see that inside the tarball, there are lots of directories and some json files. Each directory is basically one layer in the dockerfile.

We can also export the whole container using `docker export`, and the resulting tar will contain the actual filesystem.
```bash
adhy$ docker export webserver -o nginx-export.tar
adhy$ tar -xvf nginx-export.tar --one-top-level
adhy$ cd nginx-export.tar
bin boot dev ... # we can see the filesystem
```

# Docker Client and Docker Engine
When we type `docker` on our command line, we are invoking the Docker Client. Docker Client communicates with the Docker Engine, which is a service which actually manages the containers.

The docker client talks to the engine through a Linux socket file (which is just a way of exposing a service locally without having to listen on the network) over HTTP REST API. We can see these by listening to the socket.

```bash
# get the docker socket and connect it to a temporary socket
adhy$ sudo socat -v UNIX-LISTEN:/tmp/tempdock.sock,fork UNIX-CONNECT:/var/run/docker.sock

# tell docker to connect to a different socket (our temporary socket)
# and then list the images
adhy$ sudo docker -H unix:///tmp/tempdock.sock images

# in our socat terminal, we can see the HTTP request sent by the Docker Engine
```

# Bonus: Container Security
This is "The most pointless Docker command ever”, which basically gets us a root shell on the host. Since containers are basically processes running on a machine, if we strip away all the security layers and namespaces, we simply get access to the host itself.

```bash
docker run -ti --privileged --net=host --pid=host --ipc=host --volume /:/host busybox chroot /host
# -ti                : give a terminal
# --privileged       : remove all security layers
# --net, --pid, --ipc: remove all the namespaces. Don't give the container a namespace resource, use HOST resource, etc.
# --volume           : mount the host's root file system inside the directory called /host
# busybox            : the name of the container
# chroot /host       : the command to be run by docker
```

# References
["An Introduction to Container Hacking"](https://www.youtube.com/watch?v=udT99u-rTOQ) by Rory McCune (Aqua Security)
