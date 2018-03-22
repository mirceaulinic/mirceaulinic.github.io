---
layout: post
title: Getting started with Salt for network automation, using Docker
subtitle: The easy way
bigimg: /img/docker.jpg
---

This is going to be a very short blog post, but hopefully it will provide an
easier intro for the (network) engineers that want to get get started with Salt
to leverage its superpowers for network automation.

One of the easiest approaches is using Docker - even though you don't know a lot
about containers yet, just executing some commands should setup everything for
you.

To show the use-case, I will have a very simple example and manage two devices
using Salt Proxy Minions. We will thus need the following ingredients:

- A container for the Salt Master.
- One container for each device managed.

The Salt Master container
-------------------------

It is a good practice to ensure that there is no version mismatch between the
Master and the Minions connected. So I built an image which you can just use
and spawn a new container. The image is available at 
[mirceaulinic/salt-master](https://hub.docker.com/r/mirceaulinic/salt-master/)
and can be used straight away:

```bash
$ docker run -d --name salt-master -it mirceaulinic/salt-master:2017.7.4
76cdde4d645de7471451b882d1299933eb73bab390db3d6d65ff774a83c58364
```

Executing this command, it starts a new container identified by the name
``salt-master`` and ID ``76cdde4d645de7471451b882d1299933eb73bab390db3d6d65ff774a83c58364``
(or ``76cdde4d645d``). And now you have a Salt Master running on your machine,
and - bonus - together with the modules available in the next Salt release,
Oxygen.

To get into the container for the Salt Master you can execute:

```bash
$ docker exec -it salt-master bash
root@76cdde4d645d:/etc/salt#
```

In order to have data available for the 2 Proxy Minions that we want to connect
to this Master, we will need to ensure that Pillars are available:

1. Create the Pillar Top file mapping:

   ``/etc/salt/pillar/top.sls``:
   ```yaml
   base:
     device1:
       - device1_pillar
     device2:
       - device2_pillar
    ```

    This structure instructs Salt to compile the Pillar data for the Minion
    ``device1`` using the contents from ``device1_pillar.sls``, and for
    ``device2`` from ``device2_pillar.sls``.

2. Set the contents of the device Pillar files

``/etc/salt/pillar/device1_pillar.sls``:
```yaml
proxy:
  proxytype: napalm
  driver: junos
  ip: 10.10.10.1
  username: salt
  password: 5@1tisbliss
```

Note: If you don't have a network device to connect to (or VM), to start however
the container for the Proxy Minion, you can use the ``dummy`` Salt Proxy module,
e.g:

```/etc/salt/pillar/device1_pillar.sls``:
```yaml
proxy:
  proxytype: dummy
```

The Salt Proxy Minion containers
--------------------------------

As mentioned above, we aim to manage two remote devices, hence we will need to
run two separate Proxy Minion processes. Similarly, I have already prepared
another Docker image ([mirceaulinic/salt-proxy](https://hub.docker.com/r/mirceaulinic/salt-proxy))
which again can just be used:

```bash
$ docker run -d --name device1 -e PROXYID=device1 -e LOG_LEVEL=debug --link salt-master -it mirceaulinic/salt-proxy:2017.7.4
507a07aebd95b819a2868febc5a0fa774f0ab38bde5f873ae1b1590742b59b76
```

The command above, when executed, will start a container with the name
``device1` using the image ``mirceaulinic/salt-proxy:2017.7.4`` to which we
send the environmental variables ``PROXYID``, which is the ID of the Proxy
Minion running in this Docker container (which many be different than the name
of the container itself), ``LEVEL_LOG`` - optional - is the logging level. If
you are starting to play with the Salt Proxy for the very first time, you may
want to have the logging level set to ``debug`` to see the entire output. The
logged messages can be checked using ``docker logs <container name or ID>``,
e.g., ``docker logs device`` or ``docker logs 507a07aebd95``.

The other option ``--link salt-master`` is very important, as Docker will create
a bridge between the two containers, so they can communicate. There are other
(perhaps better approaches), but this is beyond the scope of this post and a bit
more tedious, though they are flexible enough to be applied in any other setup.
You can read more about that in ---TODO---

With these said, we started the first container and we should see the following
messages into the logs:

```
[ERROR   ] The Salt Master has cached the public key for this node, this salt minion will wait for 10 seconds before 
attempting to re-authenticate
[INFO    ] Waiting 10 seconds before retry.
[ERROR   ] The Salt Master has cached the public key for this node, this salt minion will wait for 10 seconds before 
attempting to re-authenticate
[INFO    ] Waiting 10 seconds before retry.
```

The user that has already played with Salt would instantly notice that we need
to go on the Master side and accept the Minion key of ``device1``:

```bash
root@5e21961a8ca2:~# salt-key -y -a device
The following keys are going to be accepted:
Unaccepted Keys:
device1
Key for minion device1 accepted.
```

The Proxy should be up immediately after, and we can check that byexecuting a
simple function, e.g., ``test.ping``:

```bash
root@5e21961a8ca2:~# salt device1 test.ping
device1:
    True
```

Note: 

The very easy way
-----------------


