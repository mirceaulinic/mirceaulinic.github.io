---
layout: post
title: Network automation using Salt for large scale deployments
bigimg: /img/scale.jpg
---

Very often I refer to Salt as one of the most suitable tools for managing large
scale deployments - either servers or network devices, but not limited to.
While on a server one can install the regular Minion directly, unfortunately,
we don't have this luxury and flexibility to do the same on traditional network
gear. For this reasoning, we have the Proxy Minion, which is a process capable
to run anywhere, as long as: i. it is able to contact the Master, and ii. it
can connect to the remote device. One Proxy Minion per device. The theory
sounds good so far.

The challenge
=============

As mentioned above, and it is good to remember that for one device managed, you
need to startup a Proxy Minion process. Without requiring a lot of forward
thinking, the reader will notice that for 1000s or network devices you need to
start and maintain 1000s of processes. And that's not particularly trivial. But,
as you'll see below, it can be reduced very easily to a 10 liner Jinja and YAML
beauty with little effort.

To fully understand the potential solutions I will present below, it is good to
understand the Salt basics and be familiar with different concepts such as
SLS, PIllar or State. Seth House and I have expanded on these topics (and others
) in our
[Network Automation at Scale](http://www.oreilly.com/webops-perf/free/network-automation-at-scale.csp)
book published at O'Reilly.

In the examples below, I will assume that you have a source of truth (either a
database which you can use to ingest data into Salt via the External Pillar, or
simply a plain SLS Pillar as a file, etc.). Or, to put this differently, I will
just assume that under the ``devices`` (can be anything, of course) Pillar
field you have the list of devices to be managed.

A Partial Solution
------------------

A very frequent approach is starting up all the processes on a single server. It
is perhaps the least optimal among the solution I included in this post, but the
easiest to understand and implement

A Proxy Minion in general consumes about 60MB of memory, though this can go
sometimes beyond 80MB per process for SSH-based Proxies for which we didn't
find yet a multiprocessing solution (SSH-based Proxies are still limited to
multithreading - the root cause goes somewhere into a library named Paramiko,
and the subject is too broad to expore it right now). This is the price paid
to receive the blessing of the event-driven automation, PCI compliance for free,
a very flexible and powerful templating language and so on.

Anyway, either 60MB or 80MB of memory is not that much, and RAM is incredibly
cheap nowdays. For example, if your network has 100 nodes, you only need 8GB of
RAM, which - I assume any decent server should have (as a matter of fact, my
old MacBook has 16GB RAM which could easily be used to automate a small to
medium sized network, and I do know global networks much smaller than that).

# Ingredients

1. The regular Minion to be installed and running on the server where the
   processes will be running.
2. (Derived from 1) Executing ``salt-call pillar.get devices`` on the regular
   Minion provides the list of devices to be managed.
3. The 
   [``service.running``](https://docs.saltstack.com/en/latest/ref/states/all/salt.states.service.html#salt.states.service.running)
   State function.

Leveraging the power of the SLS, the state is as simple as:

```sls
{%- for device in devices %}
Startup the Proxy for {{ device }}:
  service.running:
    - name: salt-proxy@{{ device }}
{%- endfor %}
```

Save this into an SLS file, say ``/etc/salt/states/start_proxies.sls``,
then all the Proxies can be started by simply executing
``salt-call state.apply start_proxies``. This is the equivalent of you running
N times ``systemctl start salt-proxy@<device_name>`` from the CLI of the server
- ``N`` is the number of devices, and ``salt-proxy@<device_name>`` is the
systemd service that manages the Proxy process for ``device name``.

That's all. And both the command you execute and the content of the SLS will
forever be the same regardless on how many devices you have: as data (which goes
into the Pillar) is decoupled from the automation logic (into the
State), you'll only work with data from here on, i.e,, just update your source
of truth.

Distributed Proxies
-------------------

While running all the Proxies on a single server can work fine, it does have
the obvious limitation that you can't run 1000s of processes on the same box,
for a variety of reasons: memory, single point of failure, ports, and so on.

The approach above can be used to distribute the Proxies on multiple servers
(not just one), by inserting an additional check into the ``start_proxies.sls``
State. This depends very much on your environmental constraints, the type of
the network, density of devices, the type of connection, and many other angles.
There is no general solution, it _really_ depends on you to figure out what's
the most suitable way for your network(s) to spread the Proxies out.

Docker containers
-----------------

Again, here I see two nice possibilities - centralised and distributed.
As mentioned in the beginning of this port, the Proxies present that advantage
that they are just processes able to run anywhere. Including Docker containers.

# Starting many Docker containers

# Container orchestrator: Kubernetes

Which one is the best? It depends. Again, this is multi-dimensional problem
that doesn't have a single answer, but instead you will need to evaluate which
suits the best your needs, and fullfills the requirements.

