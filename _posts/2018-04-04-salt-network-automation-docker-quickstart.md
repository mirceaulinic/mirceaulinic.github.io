---
layout: post
title: Getting started with Salt for network automation, using Docker
subtitle: The easy way
bigimg: /img/whale.jpg
---

This is going to be a very short blog post, but hopefully it will provide an
easier intro for the (network) engineers that want to get started with Salt
to leverage its superpowers for network automation.

One of the easiest approaches is using Docker - even though you may not know a
lot about containers yet, just executing some commands should set up everything
for you.

We will need the following ingredients:

- A container for the Salt Master.
- One container for each device managed.

The Salt Master container
-------------------------

It is a good practice to ensure that there is no version mismatch between the
Master and the Minions connected. So I built an image which you can use
and start a container for the Salt Master. The image is available at 
[mirceaulinic/salt-master](https://hub.docker.com/r/mirceaulinic/salt-master/)
and can be used straight away:

```bash
$ docker run -d --name salt-master -it mirceaulinic/salt-master:2017.7.5
76cdde4d645de7471451b882d1299933eb73bab390db3d6d65ff774a83c58364
```

Executing this command, it starts a new container identified by the name
``salt-master`` and ID ``76cdde4d645de7471451b882d1299933eb73bab390db3d6d65ff774a83c58364``
(or ``76cdde4d645d``). And now you have a Salt Master running on your machine.

To get into the container for the Salt Master you can execute:

```bash
$ docker exec -it salt-master bash
root@76cdde4d645d:/etc/salt#
```

The Salt Proxy Minion containers
--------------------------------

As mentioned above, for each device we aim to manage we need a separate
container started.. Similarly, I have already prepared
another Docker image ([mirceaulinic/salt-proxy](https://hub.docker.com/r/mirceaulinic/salt-proxy))
which again can just be used:

```bash
$ docker run -d --name proxy-device1 -e PROXYID=device1 -e LOG_LEVEL=debug -it mirceaulinic/salt-proxy:2017.7.5
507a07aebd95b819a2868febc5a0fa774f0ab38bde5f873ae1b1590742b59b76
```

The command above, when executed, will start a container with the name
``proxy-device1`` using the image ``mirceaulinic/salt-proxy:2017.7.5`` to which
we send the environment variables ``PROXYID``, which is the ID of the Proxy
Minion running in this Docker container (which many be different than the name
of the container itself), and ``LEVEL_LOG`` - optional - is the logging level.
If you are starting to play with the Salt Proxy for the very first time, you may
want to have the logging level set to ``debug`` to see the entire output. The
logged messages can be checked using ``docker logs <container name or ID>``,
e.g., ``docker logs device`` or ``docker logs 507a07aebd95``.

The very easy way
-----------------

Even though they're simple, executing these commands manually ain't fun, which
is why Docker created [Docker Compose](https://docs.docker.com/compose/) to
manage multiple containers.

In this scope I created a Docker Compose file, which not only that will start
the containers you want, but also mount the files required for each:

```yaml
version: '2.1'

services:
  salt-master:
    image: mirceaulinic/salt-master:2017.7.5
    hostname: salt-master
    container_name: salt-master
    environment:
      - LOG_LEVEL
    volumes:
      - ./master:/etc/salt/master
      - ./pillar/:/etc/salt/pillar/
      - ./states/:/etc/salt/states/
  salt-proxy:
    image: mirceaulinic/salt-proxy:2017.7.5
    hostname: ${PROXYID}
    container_name: salt-proxy-${PROXYID}
    volumes:
      - ./proxy:/etc/salt/proxy
    environment:
      - LOG_LEVEL
      - PROXYID
    depends_on:
      - salt-master
```

Executing the Docker Compose file above (e.g., ``docker-compose up -d``), it
will start two containers, and it will mount the volumes for the Pillar, States,
and the master/proxy configuration file. The Proxy Minion container requires the
``PROXYID`` environment variable, so the easiest way to send it to the container
in the case would be thorugh a Makefile:

```make
all:
    docker-compose up -d
```

I put all of these (including the Dockerfiles for the Master and the Proxy
Minion) in a [GitHub repository](https://github.com/mirceaulinic/salt-netdev-docker)
which you can clone and start playing with Salt just straight away:

```bash
$ git clone git@github.com:mirceaulinic/salt-netdev-docker.git && cd salt-netdev-docker
```

Inside the ``salt-netdev-docker`` we only need to execute the Makefile, passing
the ``PROXYID`` variable with the ID of the Minion we want to start:

```bash
$ make PROXYID=dummy
docker-compose up -d
Creating salt-proxy-dummy ...
Creating salt-master ...
Creating salt-proxy-dummy
Creating salt-master ... done
```

Executing the command above, it will start a container for the Master and another
one for the ``dummy`` Proxy, as configured in the docker-compose file (service
``salt-master`` and ``salt-proxy``, respectively). As defined into the
[``dummy_pillar.sls`` Pillar SLS 
file](https://github.com/mirceaulinic/salt-netdev-docker/blob/master/pillar/dummy_pillar.sls),
the ``dummy`` Proxy will use the [``dummy`` Proxy module](https://docs.saltstack.com/en/latest/ref/proxy/all/salt.proxy.dummy.html)
, specially designed for testing.

We can notice that the containers are up and running:

```bash
$ docker ps -a
CONTAINER ID        IMAGE                               COMMAND                   CREATED             STATUS              PORTS               NAMES
180efa8f962b        mirceaulinic/salt-master:2017.7.5   "/bin/sh -c \"/usr/..."   35 minutes ago      Up 35 minutes       4505-4506/tcp       salt-master
1659f0ed5f23        mirceaulinic/salt-proxy:2017.7.5    "/bin/sh -c \"/usr/..."   35 minutes ago      Up 35 minutes                           salt-proxy-dummy
```

To check that everything works well, we can get into the Master container and
execute a test state:

```bash
$ docker exec -it salt-master bash
root@salt-master:/# salt dummy state.apply test
dummy:
----------
          ID: Yay it works
    Function: test.succeed_without_changes
      Result: True
     Comment: Success!
     Started: 11:57:21.295204
    Duration: 0.76 ms
     Changes:

Summary for dummy
------------
Succeeded: 1
Failed:    0
------------
Total states run:     1
Total run time:   0.760 ms
root@salt-master:/# %
```

The ``test`` state SLS is defined in the same repository, under the ``states``
directory, mounted to the container:
[https://github.com/mirceaulinic/salt-netdev-docker/blob/master/states/test.sls](https://github.com/mirceaulinic/salt-netdev-docker/blob/master/states/test.sls)
(as defined in the Docker Compose file).

One important detail to keep in mind is that Docker Compose by default will
create a dedicated network for the project (the project name defaulting to the
directory name). In this case, as the directory name is ``salt-netdev-docker``
(if cloned from my repo), the network will be named ``saltnetdevdocker_default``
using the ``bridge`` driver. To be able to communicate externally, and manage
devices from elsewhere, you may want to read the
[Docker Compose Networking documentation](https://docs.docker.com/compose/networking/)
and eventually configure other network(s) depending on your own use case.

Additionally, remember to update the Pillar Top file (found under
[``pillar/top.sls``](https://github.com/mirceaulinic/salt-netdev-docker/blob/master/pillar/top.sls))
before starting a container for a Proxy Minion with another ID than ``dummy``,
as well as the appropriate Pillar SLS file(s). The dependencies for NAPALM are
already installed into the ``salt-proxy`` image, so you should be able to start
a [NAPALM](https://docs.saltstack.com/en/latest/ref/proxy/all/salt.proxy.napalm.html)
as well as a [Junos](https://docs.saltstack.com/en/latest/ref/proxy/all/salt.proxy.junos.html)
Proxy Minion.

I put these together hoping to ease the first steps when getting started with
Salt. While my playground may equally be used in a production environment (up to
an extent), I definitely enourage you to consider disabling the 
[``open_mode``](https://docs.saltstack.com/en/latest/ref/configuration/master.html#open-mode)
before doing so, as well as other security aspects. As always, you are welcome
[hit me up on Twitter](https://twitter.com/mirceaulinic), or on the
[networktocode](https://networktocode.slack.com) and
[SaltStack Community](https://saltstackcommunity.slack.com) Slack channels.
