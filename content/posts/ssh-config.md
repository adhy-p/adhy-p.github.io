+++ 
date = 2022-08-30T11:11:46+08:00
title = "Playing with SSH config"
description = ""
slug = ""
authors = []
tags = ["writeup"]
categories = []
externalLink = ""
series = []
+++

## Motivation

When I was taking Parallel Computing course in NUS, I had to ssh to the school's server to do my work. The server is located on the school's private network, so I had to use a jump host to connect to the server.
```
ssh username@soctf-pdc-001.comp.nus.edu.sg -J username@student.comp.nus.edu.sg
```

The command was super long, so I created a ssh config file:
```
Host student
    User username
    Hostname student.comp.nus.edu.sg
    ForwardAgent yes
    Forward X11 yes

Host parallel
    User username
    Hostname soctf-pdc-001.comp.nus.edu.sg
    ProxyJump student
```

Now I can just type this to connect to the server:
```
ssh parallel
```
A big part of the course was to benchmark the parallel programs and compare it to the standard, sequential implementation. We had to experiment using different number of cores, and sometimes using different clockspeed. In other words, the programs had to be benchmarked using different hardwares. Since I need to ssh to multiple servers to do the benchmarking, I need to create a new entry for each server in my ssh config file. Soon enough, my ssh config file became really cluttered and ugly:

```
... more entries ...

Host parallel1
    User username
    Hostname soctf-pdc-001.comp.nus.edu.sg
    ProxyJump student

Host parallel2
    User username
    Hostname soctf-pdc-002.comp.nus.edu.sg
    ProxyJump student

Host parallel3
    User username
    Hostname soctf-pdc-003.comp.nus.edu.sg
    ProxyJump student

... more entries ...
```

If it was only 2 or 3 server, it would be perfectly fine. But, we had a total of **24** servers... Can we do better?

## First attempt
Scrolling through `man ssh_config(5)`, I noticed something interesting:
```
... more entries ...

PATTERNS
     A pattern consists of zero or more non-whitespace characters, ‘*’ (a wildcard that matches zero or more characters), or ‘?’ (a wildcard that matches exactly one character).  For example, to specify a set of declarations for any
     host in the ".co.uk" set of domains, the following pattern could be used:

           Host *.co.uk

     The following pattern would match any host in the 192.168.0.[0-9] network range:

           Host 192.168.0.?

... more entries ...

TOKENS
     Arguments to some keywords can make use of tokens, which are expanded at runtime:

           %%    A literal ‘%’.
           %C    Hash of %l%h%p%r.
           %d    Local user's home directory.
           %f    The fingerprint of the server's host key.
           %H    The known_hosts hostname or address that is being searched for.
           %h    The remote hostname.
           %I    A string describing the reason for a KnownHostsCommand execution: either ADDRESS when looking up a host by address (only when CheckHostIP is enabled), HOSTNAME when searching by hostname, or ORDER when preparing the
                 host key algorithm preference list to use for the destination host.
           %i    The local user ID.
           %K    The base64 encoded host key.
           %k    The host key alias if specified, otherwise the original remote hostname given on the command line.
           %L    The local hostname.
           %l    The local hostname, including the domain name.
           %n    The original remote hostname, as given on the command line.
           %p    The remote port.
           %r    The remote username.
           %T    The local tun(4) or tap(4) network interface assigned if tunnel forwarding was requested, or "NONE" otherwise.
           %t    The type of the server host key, e.g.  ssh-ed25519.
           %u    The local username.

... more entries ...
```

Using this information (and after scrolling through a lot of online forums), I came up with this config file:
```
Host student
    User username
    Hostname student.comp.nus.edu.sg
    ForwardAgent yes
    Forward X11 yes

Host soctf-pdc-*
    User username
    Hostname %h
    ProxyJump student
```

By utilizing `*` pattern, the configuration (`User`, `Hostname`, and `ProxyJump`) will be applied to the current connection if the hostname starts with `soctf-pdc-`. Since we want to connect to the host we specify when we type `ssh HOST`, the `%h` token is used in the `Hostname` field.
Now, we type the following to connect to the servers:
```
ssh soctf-pdc-001.comp.nus.edu.sg # to connect to the first server
ssh soctf-pdc-002.comp.nus.edu.sg # to connect to the second server
...
ssh soctf-pdc-024.comp.nus.edu.sg # to connect to the last server
```
Since the `soctf-pdc-*` servers and the `student` server are in the same network, we can even omit the domain name.
```
ssh soctf-pdc-001
ssh soctf-pdc-002
...
ssh soctf-pdc-024
```
Not bad, but the command is still rather long. Again, can we do better?

## Second attempt
We can do something like this to make it shorter:
```
Host pdc-*
    User username
    Hostname soctf-%h
    ProxyJump student
```
Now, we can type the following to ssh to the servers:
```
ssh pdc-001
ssh pdc-002
...
ssh pdc-024
```

Sweet! It worked. But, now I am curious. If I were to ssh to the servers by typing `ssh soctf-*` instead of `ssh pdc-*`, how can I do that?

Extracting the number at the end of the supplied hostname and append it to the hardcoded hostname should work right?

```
Host soctf-*
    User username
    Hostname soctf-pdc-`echo %h | cut -d'-' -f2`
    ProxyJump student
```
Well, unfortunately it doesn't:
```
adhy:~$ ssh soctf-001
/home/adhy/.ssh/config line 39: keyword hostname extra arguments at end of line
/home/adhy/.ssh/config: terminating, 1 bad configuration options
```
## Exploring ProxyCommand 
After some googling and scrolling through the manual page, I came across another option called `ProxyCommand`:
```
ProxyCommand
    Specifies the command to use to connect to the server.  The command string extends to the end of the line, and is executed using the user's shell ‘exec’ directive to avoid a lingering shell process.

    Arguments to ProxyCommand accept the tokens described in the TOKENS section.  The command can be basically anything, and should read from its standard input and write to its standard output.  It should eventually connect an sshd(8) server running on some machine, or execute sshd -i somewhere.  Host key management will be done using the Hostname of the host being connected (defaulting to the name typed by the user).  Setting the command to none disables this option entirely.  Note that CheckHostIP is not available for connects with a proxy command.

    This directive is useful in conjunction with nc(1) and its proxy support.  For example, the following directive would connect via an HTTP proxy at 192.0.2.0:

    ProxyCommand /usr/bin/nc -X connect -x 192.0.2.0:8080 %h %p
```
Let's give it a try. Can we replicate our very first command

`ssh soctf-pdc-* -J student.comp.nus.edu.sg` using `ProxyCommand`?
```
Host soctf-*
    User e0407797
    ProxyCommand ssh soctf-pdc-`echo %h | cut -d'-' -f 2` -J student
```
Sadly, the answer is no:
```
adhy:~$ ssh soctf-001
Pseudo-terminal will not be allocated because stdin is not a terminal.
-bash: line 1: $'SSH-2.0-OpenSSH_9.1\r': command not found
```

After googling the error message, I added the `-t` flag. It managed to connect to the server, but I didn't get a shell.

```
adhy:~$ ssh -vvv soctf-012
OpenSSH_9.1p1, OpenSSL 3.0.7 1 Nov 2022
debug1: Reading configuration data /home/adhy/.ssh/config
debug1: /home/adhy/.ssh/config line 37: Applying options for soctf-*
debug1: Reading configuration data /etc/ssh/ssh_config

... more entries ...

debug1: Executing proxy command: exec ssh -t soctf-pdc-`echo soctf-001 | cut -d'-' -f 2` -J student

... more entries ...

debug1: Local version string SSH-2.0-OpenSSH_9.1
Pseudo-terminal will not be allocated because stdin is not a terminal.

we are connected!! --> debug1: kex_exchange_identification: banner line 0: Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-135-generic x86_64)

... more entries ...

```
After another round of googling, I came across the `-W` flag from ssh. From man ssh(1):
```
 -W host:port
    Requests that standard input and output on the client be forwarded to host on port over the secure channel.  Implies -N, -T, ExitOnForwardFailure and ClearAllForwardings, though these can be overridden in the configuration file or using -o command line options.
```
Looks convincing. Let's try it
```
Host soctf-*
    User username
    ProxyCommand ssh student -W soctf-pdc-`echo %h | cut -d'-' -f 2`:%p
```
```
adhy:~$ ssh soctf-001
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-135-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

...

Last login:
username@soctf-pdc-001:~$
```

Nice!

## Conclusion
In this post, we explored how to use combinations of pattern, token, and ProxyCommand to easily connect to multiple hosts. While it might be better for us to type the whole hostname `soctf-pdc-*` and use the `%h` token directly (first attempt), we now know that we can do with `ProxyCommand` and this knowledge might be useful in the future when we are dealing with more servers with more complex hostnames.

Last updated: May 2023
