---
layout: post
title: Running Salt Proxy Minions on OpenBSD
subtitle: The beginning of the OpenBSD era
---

As I have previously attempted several times in the past, I am (finally) very
close to switch to OpenBSD, a more stable and reliable operating system that I
like. Before starting to make the actual change on my work machine I thought
about testing some of the tools I'm currently using and know what to expect.

Installation and Startup
========================

With these said, I started playing with Salt, and it was simple and straight
forward. First step - install Salt: ``pkg_add salt``. This will bring several
ports for Python futures, ZeroMQ, or Tornado which are needed for Salt.

After configuring the ``pillar_roots`` in the ``/etc/salt/master`` config file
for the Master, I started up the master process using
[``rcctl``](https://man.openbsd.org/rcctl):

```bash
# rcctl start salt_master
salt_master(ok)
```

Configuring the ``/etc/salt/minion`` config file so the Minion connects to the
local Salt Master (i.e., ``master: localhost``), the Minion service starts
successfully:

```bash
# rcctl start salt_minion
salt_minion(ok)
```

And we can verify this from the CLI:

```bash
# salt-call grains.get os
local:
    OpenBSD
# salt-call grains.get osrelease
local:
    6.2
```

Nothing different so far.


Starting up the Proxy Minions
=============================

The Salt package for OpenBSD comes with the rc file for salt-proxy as well:
``/etc/rc.d/salt_proxy``.

While typically you run a single regular Minion on a given machine, it is very
like that there are multiple Proxy processes. Additionally, the default Salt rc
file has the following configuration for the ``salt-proxy`` daemon:

```bash
daemon="/usr/local/bin/salt-proxy -d"
```

But this is not enough to start a proxy, as the ``proxyid`` argument is
mandatory:

```bash
# salt-proxy -d
Usage: salt-proxy [options]

salt-proxy: error: salt-proxy requires --proxyid
```

With these said, we need to send somehow the ``proxyid`` argument. Reading the
``rcctl(8)`` manual, it explains what to do in cases like that:

> The recommended way to run a second copy of a given daemon for a different
> purpose is to create a symbolic link to its rc.d(8) control script:

> # ln -s /etc/rc.d/snmpd /etc/rc.d/snmpd6 
> # rcctl set snmpd6 status on 
> # rcctl set snmpd6 flags -D addr=2001:db8::1234 
> # rcctl start snmpd6

Our needs are very similar in this case, so I did the same:

```bash
# ln -s /etc/rc.d/salt_proxy /etc/rc.d/salt_proxy_dummy
# rcctl set salt_proxy_dummy status on
# rcctl set salt_proxy_dummy flags --proxyid dummy
```

After starting the service and accepting the key, the Proxy is up and running:

```bash
# rcctl start salt_proxy_dummy
salt_proxy_dummy(ok)
```

```bash
# salt dummy test.ping
dummy:
    True
```

Starting many Proxy Minions
---------------------------

I have managed to startup a Proxy Minion, but what about many? Executing the
three commands above for each and every device is tedious and cannot scale very
well. I thus have figured the following way:

1. Have a separate rc file per Proxy, each having the daemon instruction
   explicitly specifying its Minion ID:

``/etc/rc.d/salt_proxy_test`` (excerpt):

```bash
daemon="/usr/local/bin/salt-proxy -d --proxyid test"
```

2. Start the service (using the regular Minion that controls the machine where
   the Proxy processes are running):

```bash
# salt-call service.start salt_proxy_test
local:
    True
```

Extending the same to a (very) large number of Proxy Minions, you can easily
manage the rc files and start the services using a state executed on the regular
Minion:

1. Using the [``file.managed``](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.file.html#salt.states.file.managed)
   State function to generate the contents of the rc file for each Proxy, with its
   own Minion ID.
2. Using the [``service.running``](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.service.html#salt.states.service.running)
   State function start the service.

These two steps would suffice to start an arbitrary number of Proxy Minions, and
the command executed will always be the same regardless how many processes you
aim to manage.

Conclusions
===========

I am still a novice when it comes to OpenBSD, I have plenty to learn, but it
looks like the transition will be much smoother than I expected. I am already
looking forward to the handover, and, hey, I will no longer be running
[systemd](https://twitter.com/systemdsucks). :-)
