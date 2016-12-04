---
layout: post
title: "Connect to an SSH server until it works"
date: 2016-12-03 21:00:00 -0600
categories: shell
---
I've come across a small annoyance with SSH when waiting for a server to boot.
For instance, while starting a [Karaf][karaf] container, rebooting a Linux VM,
or waiting for an AWS instance to come online, I don't like having to continually
retry to connect to the server. Instead, I've been using a small little Bash
function for a while now to continually retry to connect to the server. What's
neat about how this works is that if you get disconnected, it will continue to
retry. However, if you disconnect manually, it will stop retrying because SSH
exits with a successful status when you disconnect properly. The following
function can be used with any implementation of SSH, but I recommend using
[OpenSSH][ssh].

```bash
function ssh-auto-retry()
{
    false
    while [ $? -ne 0 ]; do
        ssh "$@" || (sleep 1;false)
    done
}
```

This function can be used to automatically retry an SSH connection as described
above. There is also a related configuration option in OpenSSH that can be used
to retry connections a finite number of times. Set the `ConnectionAttempts`
option to a number larger than 1 to automatically retry a connection up to that
many times. What's neat about this option is that you can set it per host. For
example, we can set up automatic connection retries on our local Karaf server
with the following `~/.ssh/config` snippet:

```
Host karaf
HostName localhost
Port 8101
User karaf
ConnectionAttempts 1000
```

More info about available OpenSSH options are in the [ssh\_config(5)][man] man
page.

---

Note that we can refactor the above `ssh-auto-retry` function to make a generic
`auto-retry` function for retrying shell commands until they succeed. This uses
a one second delay between each attempt, but an exponential backoff decay could
be added using some basic math. First, here's a simple `auto-retry` command:

```bash
function auto-retry()
{
    false
    while [ $? -ne 0 ]; do
        "$@" || (sleep 1;false)
    done
}
```

Then, we could easily rewrite `ssh-auto-retry` as follows:

```bash
function ssh-auto-retry()
{
    auto-retry ssh "$@"
}
```

As for an exponential backoff version, we can reimplement `auto-retry` by doubling
the timeout every retry:

```bash
function auto-retry()
{
    let backoff=1
    false
    while [ $? -ne 0]; do
        "$@" || (sleep $((backoff*=2));false)
    done
}
```

In theory, though, the `backoff` variable will eventually overflow, but it would
take many years of sleeping before that would happen!

[karaf]: https://karaf.apache.org/
[man]: http://man.openbsd.org/ssh_config
[ssh]: https://www.openssh.com/
