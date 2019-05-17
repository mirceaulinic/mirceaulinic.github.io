---
layout: post
title: A (Proxy) Minion-less Approach for Network Automation using Salt
---

Salt is currently one of the largely adopted automating frameworks, and perhaps
one of the most complete and flexible. But these - including support for
several types of topologies, large cardinal of native capabilities and features,
ability to easily extend it in your own environment, and so on - come with a
cost. And that is setting up and maintaining an infrastructure to facilitate and
leverage these capabilities.

In the networking space, Salt has had native support for automating networks
beginning with release 2016.11.0 (codename Carbon) and these have been
continuously expanded since, now covering a considerable number of platforms -
either through [NAPALM](https://github.com/napalm-automation/napalm) (and the
available [community drivers](https://github.com/napalm-automation-community/)),
or various others natively integrated modules. Network gear is generally managed
through Proxy Minions, which is a derivative of regular Salt Minion allowing to
manage the targeted device remotely. Let's make one step back: as you probably
know very well already, a regular Minion is a software managing the machine it
is running on; but the issue here is that you generally cannot install and run
custom software on legacy network gear, therefore you cannot install the regular
Minion on the network device and manage it like that. That said, the remaining
solution was to create a different flavour of this regular Minion, which is
called _Proxy Minion_, that is a process capable to run anywhere connecting to
the targeted device through an API or SSH (i.e., NETCONF, gRPC, HTTP API, etc.).
The problems arise when you find out that for every device you want to automate,
you have to spin up one Proxy Minion process. While this isn't terribly
complicated, it does require a certain degree of knowledge how to do this; in
fact, I have previously elaborated on this topic in a previous blog post,
[Network automation using Salt for large scale deployments](https://mirceaulinic.net/2018-09-27-network-automation-at-scale/).
In that blog post I tried to highlight that it is not _that_ complicated,
although I do agree that it sometimes might be pointless to have an always
running a process: it does make sense when you manage a network device that
is in a more dynamic network (say an edge router that you update the
configuration frequently, or run operational commands many times a day, or
susceptible to generate a high number of events per second that require
interaction; in cases like that it's probably a bad idea to re-establish a
separate connection - especially when it's over SSH - every 5 minutes or even
more frequent than that). But it certainly doesn't make sense when your target
is less dynamic or almost statical environments: think, for example, of a
service provider that only touches the CPEs every few months / years, or the
console servers, and many other examples.

I'm very glad that after years of looking into this problem, I've _finally_
found the solution. Today, I'm very excited to announce the
[``salt-sproxy``](https://github.com/mirceaulinic/salt-sproxy) package.

``salt-sproxy``
---------------

I'll firstly present what it does and then we'll take a look at how to use. The
core idea is that ``salt-sproxy`` creates a subprocess for every device matching
the target executing the requested Salt function. Under each subprocess it's
running a lightweight version of the Proxy Minion - that is, the regular Proxy
Minion start up, including compiling the Pillar, or collecting the Grains, but
without other components that wouldn't make sense in this case, i.e., Engines,
Beacons, Scheduler etc. In short, it pretty much only establishes the connection
to the remote network device, over the API of choice. As soon as the execution
of the Salt function is finished, it closes the connection and terminates the
subprocess.

``salt-sproxy`` can be very well installed on the Master, or even on your
personal computer, as it only requires one thing: be able to connect to the
network device from where you run it. Not only that you don't require any Proxy
Minions always running, but you don't actually need a Salt Master either. It's
as easy as that: install and use. It is [available on 
PyPI](https://pypi.org/project/salt-sproxy/) so you can just go ahead and
execute ``pip install salt-sproxy`` to install.

The usage is fairly simple, very similar to the syntax of the usual ``salt``
command:

```bash
$ salt-sproxy <target> <function> [<arguments] [<options>]
```

For example, the following command would retrieve the ARP table from the device
identified as ``edge1.thn.lon``:


```bash
$ salt-sproxy edge1.thn.lon net.arp
```

To have the above work fine, you will need to have the ``pillar_roots``
correctly configured into the Master configuration file (even though you may not
run a Salt Master process) -- typically ``/etc/salt/master``. In that file you
specify the paths to where Salt should load the Pillars from (or what External
Pillars to use to pull the data). Everything you know remains the exact same:
you need to have the ``proxy`` key into the Pillar providing the ``proxytype``
along with the connection credentials, as well as a Pillar ``top.sls`` file, etc.
-- the only change is that you don't need to start any Proxy processes, and the
usage syntax is slightly different.

To use a different Salt configuration file, you can specify it using the ``-c``
option, e.g.,

```bash
$ salt-sproxy edge1.thn.lon net.arp -c /path/to/salt/config
```

The good news here is that (almost) everything is exactly preserved to what you
are already used to, and you can, e.g., use the [Returner](TODO) interface to
forward the data into a different system, write the returned output into a file,
or display the data on the CLI in the format you want, and so on, e.g.,


```bash
$ salt-sproxy edge1.thn.lon net.lldp --out=json --out-file=/home/mircea/lldp.json
```

The previous example would save the LLDP neighbours in a JSON format, in a
file found at ``/home/mircea/lldp.json``.

It works in exactly the same way when you're pushing a configuration change, for
example, through the ``net.load_config``, ``net.load_template`` functions, as
well as ``state.apply`` or ``state.highstate`` to apply States or Highstates,
respectively.

Another great point - which is particularly important to me - is being able to
invoke the custom extension function you define in your own environment. For
example, let's consider the ``example.version`` function defined in the previous
post on [Extending NAPALM's capabilities in the Salt environment](TODO):

```bash
$ salt-sproxy juniper-router example.version
```

To be able to load your custom modules, as explained in the blog post, you would
need to have the correct configuration for the ``file_roots``.

So what's the catch?
--------------------

Does it sound too good to be true? Well, it is that good, and there isn't any
catch. It might not be immediately obvious what are the implications and the
benefits to using this methodology, but it made me very happy when I've finally
been able to put this together.

TODO: difference to the salt command and the need for the Roster (and examples
without the Roster). Maybe include here the "migrating from Ansible" paragraph?

TODO: explain that it dynamically injects a Runner

Does it work only with NAPALM?
------------------------------

No. You can use any of the available [Proxy modules](TODO) in the exact same way. The
``salt-sproxy`` relies on the Proxy modules to initiate the connection, but it
doesn't require the Proxy process to be running. If you have a device that isn't
currently supported by Salt natively, just write a Proxy module for it - see
[this guide](TODO) - put it into the ``salt://_proxy`` directory, and then make
sure that the module is referenced in the ``proxytype:`` Pillar field. If you're
unsure what ``salt://_proxy`` means, please check [this](TODO) article again.

Suppose we write a custom Proxy module, e.g., ``customproxy.py`` which we put
under ``/srv/salt/extmods/_proxy``. Then, to use it, when targeting a device,
e.g., ``obscure-platform``, in the Pillar you'll need to have the following
structure:

```yaml
proxy:
  proxytype: customproxy
  ~~~ connection credentials ~~~
```

To check that the data is indeed available for the ``obscure-platform`` Minion,
you can always run the following command:

```bash
$ salt-run pillar.show_pillar obscure-platform
```

Then you should be all set to give it a go:


```bash
$ salt-sproxy obscure-platform test.ping
```

I am not going to detail on this further, but you can follow the notes from
[TODO](TODO) with the minor differences described above.

Migrating from Ansible to Salt and salt-sproxy
----------------------------------------------

If you're already an Ansible user, ``salt-sproxy`` should make it even easier to
migrate to using Salt. This is thanks to 

TODO


Even-driven automation? Not a problem
-------------------------------------

TODO: link to the dynamically injected Runner that can be used, though requires
an always running Master.


Does it work on Windows?
------------------------

I don't know. I haven't tried, as I don't have a Windows machine available, and,
to be fair, I have no idea how to use Python on Windows. But please go ahead and
give it a try, then let me know. Normally, if you're able to install ``salt``
from PyPI, ``salt-sproxy`` should be usable too - that is my guess, at least.

In the [``salt-sproxy``](https://github.com/mirceaulinic/salt-sproxy) repository
I've added a Makefile that should normally facilitate the installation on Unix
machines. If you can, please submit a PR to provide a ``.bat`` script or
anything that provides the equivalent functionality on Windows - the community
would greatly appreciate that. Thanks in advance!

Conclusions
-----------

I don't think the ``salt-sproxy`` is meant to replace the existing Proxy Minion-
based approach, but rather fill in some gaps where the Proxy Minions fall short,
or are a burden to manage. As I mentioned in the beginning of this post, in my
opinion, Proxy Minions would always make sense in dynamic environments, or
to manage devices likely to change their environment very quickly. On the long
term, I would assume that the winning combination is running both Proxy Minions
and ``salt-sproxy`` concomitantly, although there may be exceptions. It depends
very much on your operations, the network topology, what are your goals and many
other aspects on where to draw the line; I think that the rule of thumb here is
to evaluate which devices require frequent changes / interaction (for which
you would start Proxy processes), and which are more statical (which you'd
probably manage using the ``salt-sproxy``).

I hope you are going to find this helpful, and TODO.

TODO: still WIP, waiting for feedback and hopefully PRs
