+++ 
date = 2022-12-09T14:22:26+08:00
title = "Containers"
description = ""
slug = ""
authors = []
tags = ["writeup"]
categories = []
externalLink = ""
series = []
+++

Disclaimer: This is just my note for a talk ["An Introduction to Container Hacking"](https://www.youtube.com/watch?v=udT99u-rTOQ) by Rory McCune (Aqua Security)

# Containers

Containerisation is the process of packaging an application and the necessary resources (such as libraries and packages) required into one package.

The motivation behind containers is due to the myriad dependencies found in modern applications. These dependencies can:

- be difficult to install
- create difficulty for devs to diagnose and replicate faults
- conflict with each other (e.g. multiple versions of python)

Containerisation platforms make use of the `namespace` feature of the kernel, which is a feature used so that processes can access resources of the operating system without being able to interact with other processes.

Every process running on Linux will be assigned with two things: `namespace` and `PID`. Processes can only “see” other processes that are in the same namespace.

History of containers

- 1979 - chroot system call
- 2000 - FreeBSD Jails
- 2001 - Linux Vserver
- 2004 - Solaris Zones
- 2008 - LXC
- 2013 - Docker

Docker make use of existing Linux facilities to isolate each process from the underlying host and from each other.

- Process
- Namespaces
    - mount namespace: the process can only see certain directories
    - network namespace: the process can only see certain network adapters
- Capabilities: break down root rights
    - In the old days: you are either root (all access), or not root (limited access)
    - allow a little bit of what root has without giving everything
    - In container context: we give every containers some capabilities
- cgroups: restrict how much resources a process can use
    - by default, docker does not set the cgroup
    - standard fork bomb will use all the process handles on the machine and DOS the machine
- AppArmor/SEinux: mandatory access control system
    - restrict what a process can do
- Seccomp: block individual linux kernel syscalls
    - docker blocks a set of “dangerous” linux kernel syscalls by default
    - kubernetes does not implement this by default

**Containers** are just normal **processes**

Demo:

```bash
ps aux | grep -i nginx
** no result **
# run a new docker container, with name 'nameserver' with 'nginx' image
docker run -d --name webserver nginx

# from the perspective of docker, we can see that docker has an instance of nginx
docker ps
CONTAINER ID    IMAGE    COMMAND    CREATED    STATUS    PORTS    NAMES
XXXXxxxxxxxx    nginx    xxxxxxx    xxxxxxx    Up        80/tcp   webserver  

# from the host's perspective, we can see bunch of nginx processes
ps aux | grep -i nginx
root     ROOT_PID ROOT_PID ... nginx: master process nginx -g daemon off;
systemd+ PID      PID      ... nginx: worker process
systemd+ PID      PID      ... nginx: worker process
systemd+ PID      PID      ... nginx: worker process

# docker exec will execute an command inside the container
# execute touch; create a new file called 'my_new_file' in the root of the container's file system
docker exec webserver touch /my_new_file

# we can access this file by accessing the proc filesystem
sudo ls /proc/ROOT_PID/root
bin boot dev ... **my_new_file** ...
```

**A container image** are just **tarballs** with **json** metadata

Demo:

```bash
# create a tarball out of the docker nginx image
docker save nginx:latest -o nginx-save.tar

tar -xvf nginx-save.tar --one-top-level
cd nginx-save
ls
<sha256hash> # directories, one layer of dockerfile
<sha256hash> # directories
...
<sha256hash>.json
anifest.json
repositories

# using tree, we can see that each layer has another json file, an a layer.tar
```

Instead of extracting each tarball which is time consuming, we can turn any running container into tarballs

```bash
docker export webesrver -o nginx-export.tar
tar -xvf nginx-export.tar --one-top-level
cd nginx-export.tar
bin boot dev ... # we can see the filesystem
```

`Docker Client` (command line program) talks to the `Docker Engine` through a linux socket file (which is a way of exposing a service locally without having to listen on the network) over HTTP REST API. `Docker Engine` talks to the `Docker Registry` (e.g. Docker Hub) to get the images. After getting the images, `Docker Engine` runs the processes (i.e. the `containers`).

```bash
# get the docker socket and connect it to a temporary socket
sudo socat -v UNIX-LISTEN:/tmp/tempdock.sock,fork UNIX-CONNECT:/var/run/docker.sock

# tell docker to connect to a different socket (our temporary socket)
# and then list the images
sudo docker -H unix:///tmp/tempdock.sock images

# in our socat terminal, we can see the HTTP request sent by the Docker Engine
```

“The most pointless Docker command ever”

```bash
docker run -ti --privileged --net=host --pid=host --ipc=host --volume /:/host busybox chroot /host
# -ti                : give a terminal
# --privileged       : remove all security layers
# --net, --pid, --ipc: remove all the namespaces. Don't give the container a namespace resource, use HOST resource, etc.
# --volume           : mount the host's root file system inside the directory called /host
# busybox            : the name of the container
# chroot /host       : the command to be run by docker
```
