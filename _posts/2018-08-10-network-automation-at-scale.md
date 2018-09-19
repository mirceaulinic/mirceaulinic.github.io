---
layout: post
title: Network automation using Salt for large scale deployments
bigimg: /img/scale.jpg
---

Very often I refer to Salt as one of the most suitable tools for managing large
scale deployment - either servers or network devices, but not limited to.
While on a server one can install the regular Minion directly, and there are
already many well known success stories in the industry, unfortunately, we don't
have the same flexibility on traditional network gear, as the network operating
systems are generally extremely limited (to the end user).

For this reasoning, we have the Salt Proxy Minion, which is a process capable
to run anywhere, as long as: 1. it is able to contact the Master, and 2. it
can connect to the remote network device. One Proxy Minion per device. The theory
sounds good so far.

The challenge
=============

As mentioned above, and it is good to remember that for one device managed, you
need to start up a Proxy Minion process. The reader will immediately notice that
for 1000s or network devices you need to start and maintain 1000s of processes.
And this not particularly trivial... when you do it manually. :-)
But, as you'll see below, with the right approach, it can be reduced very easily
to a 10 liner Jinja and YAML with very little effort.

To fully understand the potential solutions I will present below, it is good to
understand the Salt basics and be familiar with different concepts such as
SLS, Pillar or State. Seth House and I have expanded on these topics (and others
) in our
[Network Automation at Scale](http://www.oreilly.com/webops-perf/free/network-automation-at-scale.csp)
book published at O'Reilly.

In the examples below, I will assume that you have a source of truth (either a
database which you can use to ingest data into Salt via the External Pillar, or
simply a SLS Pillar as a plain file, etc.). In other words, I will
assume that under the ``devices`` (can be any other name, of course) Pillar
key you have the data for the list of (network) devices to be managed.

Similarly, to fully understand the examples, make sure you have understood how
the location of the files suggested relates to the ``file_roots`` option on the
Master config. All the examples assume that you can have, at least, the path
``/etc/salt/states`` listed as one of the file roots in the ``base``
environment, e.g.,

```yaml
file_roots:
  base:
    - /etc/salt
    - /etc/salt/states
```

This option is important for the Salt fileserver to access the files, including
the States I will be using.

A Partial Solution
==================

A very frequent approach is starting up all the processes on a single server. It
is perhaps the least optimal among the solution I included in this post, but the
easiest to understand and implement. It is just fine as long as you don't have
a very large amount of devices to manage.

A Proxy Minion in general consumes about 60MB of memory, though this can go
sometimes beyond 80MB per process for SSH-based Proxies for which we didn't
find yet a multiprocessing solution (SSH-based Proxies are still limited to
multithreading - the root cause goes somewhere into a library named Paramiko,
and the subject is too broad to explore it right now). This is the price paid
to receive the blessing of the event-driven automation, PCI compliance for free,
a very flexible and powerful templating language and so on.

Anyway, either 60MB or 80MB of memory is not that much, and RAM is incredibly
cheap nowdays. For example, if your network has 100 nodes, you only need 8GB of
RAM, which - I assume any decent server should have. As a matter of fact, my
old MacBook has 16GB RAM which could easily be used to automate a small to
medium sized network, and I do know global networks much smaller than that; in
other words, my personal computer would suffice to manage their entire
infrastructure. :-)

Ingredients
-----------

1. The regular Minion to be installed and running on the server where the
   Proxy processes will be running.
2. (Derived from 1) Executing ``salt-call pillar.get devices`` on the regular
   Minion provides the list of devices to be managed. Again, this is up to you
   in what form, or from where to load this data (i.e., Pillar SLS file(s) or
   External Pillar, e.g., CSV file, SQL database, HTTP API etc.)
3. The 
   [``service.running``](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.service.html#salt.states.service.running)
   State function -- check the documentation.

Leveraging the power of the SLS, the State is as simple as:

```sls
{%- for device in pillar.devices %}
Startup the Proxy for {{ device }}:
  service.running:
    - name: salt-proxy@{{ device }}
{%- endfor %}
```

Save this into an SLS file, say ``/etc/salt/states/start_proxies.sls``, then all
the Proxies can be started by simply executing
``salt-call state.apply start_proxies``. This is the equivalent of you running
``N`` times ``systemctl start salt-proxy@<device_name>`` from the CLI of the server
where ``N`` is the number of devices, and ``salt-proxy@<device_name>`` is the
systemd service that manages the Proxy process for ``<device_name>``.

That's all. Both the command you execute and the content of the SLS will
forever be the same regardless on how many devices you have: as data (which goes
into the Pillar) is decoupled from the automation logic (into the
State), you'll only work with data from here on, i.e,, just update your source
of truth.

A quick example
---------------

For completeness, I will also provide an example on how is the Pillar Top file
structured, and the rest of the Pillars:

``/etc/salt/pillar/top.sls``
```yaml
base:
  my-server:
    - all_my_network_devices
  netdev1:
    - netdev1_pillar
  netdev2:
    - netdev2_pillar
```

The regular Minion managing the server where the Proxy Minions will be running
identified using the Minion ID ``my-server``, should have (at minimum) the
following Pillar data, as the list of network devices to be managed:

``/etc/salt/pillar/all_my_network_devices.sls``
```yaml
devices:
  - netdev1
  - netdev2
```

Therefore, the ``my-server`` Minion will have the following data available:

```bash
$ salt 'my-server' pillar.get devices
my-server:
    - netdev1
    - netdev2
```

Executing ``salt-call state.apply start_proxies`` on the regular Minion, or
from the Master, to invoke the ``start_proxies`` State earlier defined:

```bash
$ salt 'my-server' state.apply start_proxies
my-server:
----------
          ID: salt-proxy@netdev1
    Function: service.running
      Result: True
     Comment: Service salt-proxy@netdev1 has been enabled, and is running
     Started: 08:20:28.268712
    Duration: 791.067 ms
     Changes:
              ----------
              salt-proxy@netdev1:
                  True
----------
          ID: salt-proxy@netdev2
    Function: service.running
      Result: True
     Comment: Service salt-proxy@netdev2 has been enabled, and is running
     Started: 08:20:29.062585
    Duration: 781.164 ms
     Changes:
              ----------
              salt-proxy@netdev2:
                  True

Summary for my-server
------------
Succeeded: 2 (changed=2)
Failed:    0
------------
Total states run:     2
Total run time:   1.572 s
```

Distributed Proxies
===================

While running all the Proxies on a single server can work fine, it does have
the obvious limitation that you can't run 1000s of processes on the same box,
for a variety of reasons: not enough memory, single point of failure, limited
number of available ports, and so on.

The same approach as above can be used to distribute the Proxies on multiple
servers (not just one), by either inserting an additional check into the
``start_proxies.sls`` State, or play with the Pillar data distributed to each
Minion. This depends very much on your environmental constraints, the type of
the network, density of devices, the type of connection, and many other angles.
There is no general solution, it _really_ depends on you to figure out what's
the most suitable way for your network(s) and how to distribute the Proxy
processes on what machines and where.

I will however provide a couple of very quick examples. It pretty much boils
down to what Minions (servers) you spread the Pillar data.

A static, probably easier to follow approach
--------------------------------------------

Consider we have two servers, and we want to have the Proxies distributed as
follows: ``netdev1``, ``netdev2``, and ``netdev3`` running on ``server1``, while
``netdev4`` and ``netdev5`` on ``server2``. In that case, we could have the
Pillar Top file below:

```yaml
base:
  server1:
    - proxies_on_server1
  server2:
    - proxies_on_server2
```

While the Pillars are:

``/etc/salt/pillar/proxies_on_server1.sls``
```yaml
devices:
  - netdev1
  - netdev2
  - netdev3
```

``/etc/salt/pillar/proxies_on_server2.sls``
```yaml
devices:
  - netdev4
  - netdev5
```

The State wouldn't change in this case, as the automation logic remains the same,
only the data has changed, therefore only the target is different:
``salt -L server1,server2 state.apply start_proxies`` which would provide a
similar output as previously.

A more dynamic example
----------------------

While the above is sufficient to understand the concept, I wouldn't say it the
way to go. First of all, you might want a smarter way to have the Pillar available
everywhere than a static Top file. Secondly, the Pillar file itself doesn't scale
when it's static. A good solution is having a single Pillar SLS (surely, an
[External Pillar](https://docs.saltstack.com/en/latest/topics/development/external_pillars.html)
would work very well too), that is available to all the Minions. In that case,
the distribution of the Pillar data is made inside that SLS file. I know I keep
repeating myself, but it's good to remember that SLS is not just plain YAML, but
much, much more powerful that that. For example:

``/etc/salt/pillar/top.sls``
```yaml
base:
  'server*':
    - proxies_on_servers
```

That configuration in the Pillar Top file instructs Salt to render the
``proxies_on_servers.sls`` file and make it available to any Minion whose ID
starts with ``server``.

Consider the following Pillar SLS file:

``/etc/salt/pillar/proxies_on_servers.sls``
```sls
{%- set proxy_count = 100 %}
{%- set server_id = opts.id.replace('server', '') | int %}
{#
    get the ID of the server from the Minion ID
    server_id is therefore an integer
#}
{%- set proxy_start = (server_id - 1) * proxy_count %}
devices:
{%- for proxy_id in range(1, proxy_count+1) %}
  - netdev{{ proxy_start + proxy_id }}
{%- endfor %}
```

The above is a Pillar, yes: it provides the data to feed the ``start_proxies``
State. It is very custom for the particular case I exemplified earlier, the ID
of the server (the ``server_id`` variable) being extracted from the Minion ID
(``opts.id``). The ``for`` loop constructs the list of devices, given an
arbitrary maximum number of Proxy Minions we start per server, e.g., 100.

Using the logic above, the Pillar data for ``server1`` becomes a list of devices
from 1 to 100, while the Pillar data for ``server2`` is the list of devices
from 101 to 200, and so on:

```bash
# salt server1 pillar.get devices
server1:
     - netdev1
     - netdev2
     ... snip ...
     - netdev99
     - netdev100
#
# salt server2 pillar.get devices
server2:
    - netdev101
    - netdev101
    ... snip ...
    - netdev199
    - netdev200
```

As mentioned above, the State SLS doesn't need to change, as the automation
logic is the same, decoupled from data.

Containers
==========

Similarly to when running the Proxy Minion as system services, when managing
them as (Docker) containers we can have them centralised or distributed to run
in multiple places. The implementation is very similar, as only the state is
slightly changed (the Pillars can remain the same as previously):

``/etc/salt/states/docker_proxies.sls``
```sls
{%- for device in pillar.devices %}
Startup the Docker container for {{ device }}:
  docker_container.running:
    - name: {{ device }}
    - image: 'mirceaulinic/salt-proxy:2017.7.5'
    - binds:
      - /etc/salt/proxy:/etc/salt/proxy
    - environment:
      - PROXYID: {{ device }}
{%- endfor %}
```

This would suffice to start the Docker containers for the local or remote Proxy
Minions, when you need to manage 1, 100s or 1000s of network devices. In this
example I used the Docker image I mentioned in one of my previous
[posts](https://mirceaulinic.net/2018-04-04-salt-network-automation-docker-quickstart/),
[mirceaulinic/salt-proxy](https://hub.docker.com/r/mirceaulinic/salt-proxy/). As
the image I built expects the ``PROXYID`` environment variable, the state
configures it with the name of the network device (Proxy Minion ID). The file
``/etc/salt/proxy`` is mounted to the same path inside the container.

All of the above assume that the targeted servers have all the dependencies for
the [``docker_container`` State module](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.docker_container.html)
properly installed.

You can probably implement the same using other flavours of containers, e.g.,
LXD, or other virtualisation systems.

Kubernetes
==========

In-line with the previous paragraphs, Simon [TODO] wrote a very nice blog post
explaining how to manage your network using Kuberetes to orchestrate Proxy
Minions as Docker containers. Kubernetes is similarly built for large scale
applications, and there are plenty of good resources and blog posts on how to
achieve that.

Cloud
=====

No servers? No problem! I heard of several large organisations that have their
entire automation platform moved into the cloud. I am not going to discuss
whether this is a good idea, as it can be very subjective, and sometimes may
even be a great idea!

Salt has very good support to help you manage cloud hosts from a variety of
providers, including: Amazon AWS, Digital Ocean, Linode, Azure, HP Cloud,
Google Compute Engine, etc.

The configuration is very easy; to have the right background, you can start
reading from [https://docs.saltstack.com/en/latest/topics/cloud/](https://docs.saltstack.com/en/latest/topics/cloud/).

In this particular case I will be using [Linode](https://www.linode.com/) as my
provider of choice - check the [Salt Cloud documentation for Linode](https://docs.saltstack.com/en/latest/topics/cloud/linode.html).

I have added the following configuration:

``/etc/salt/cloud``
```yaml
start_action: state.highstate
```

``/etc/salt/cloud.providers.d/linode.conf``
```yaml
my-linode-config:
  apikey: <my-api-key>
  password: <my-password>
  driver: linode
```

``/etc/salt/cloud.profiles.d/minion1.conf``
```yaml
linode_minion1:
  provider: my-linode-config
  size: Nanode 1GB
  image: Debian 9
  location: London, England, UK
```

I have picked the smallest possible linode (Nanode 1GB) that would suffice to
manage several network devices.

Executing: ``salt-cloud -p linode_minion1 server1`` would bring up the new VM.
When executing the command from the Salt Master, the key of ``server1`` will
automatically be signed. To make sure that the Proxy Minions are started once the
new VM is up and running, have ``start_action: state.highstate`` in the cloud
config file (``/etc/salt/cloud``), and the ``start_proxies`` included in the
State Top file:

``/etc/salt/states/top.sls``
```yaml
base:
  'server*':
    - start_proxies
```

Having ``start_proxies.sls`` referenced in the State Top file means that the
State defined earlier is part of the Highstate.

With this simple setup you can start up and manage multiple network devices
without having to worry about maintaining your own infrastructure. This can be
easily scaled to hundreds / thousands VMs managed by the Salt Cloud by moving
the Cloud Profiles (i.e., the configuration from ``/etc/salt/cloud.profiles.d/``)
into the Pillar of a Salt Minion (see
[https://docs.saltstack.com/en/latest/topics/cloud/config.html#pillar-configuration](https://docs.saltstack.com/en/latest/topics/cloud/config.html#pillar-configuration))
and the [``cloud`` ](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.cloud.html)
Execution Module which can be used to spin as many instances as needed.

Which one is the best
=====================

In this post I put together a few approaches I could think of, or heard from the
community. This list of possibilities is open ended, I do not exclude that other
engineers might have though about other (better) implementations. Which one is
the best? It depends: this is multi-dimensional problem that doesn't have one
single good answer - you have to evaluate which suits the best your needs, and
fulfils the requirements. If you have another approach and you are happy to share
it with the community, please do write about it, or contact me to discuss and I
would gladly add it here.
