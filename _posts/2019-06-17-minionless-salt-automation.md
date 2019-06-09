---
layout: post
title: A (Proxy) Minion-less Approach to Network Automation using Salt
---

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "NewsArticle",
  "mainEntityOfPage": {
    "@type": "WebPage",
    "@id": "https://mirceaulinic.net/2019-06-17-minionless-salt-automation/"
  },
  "headline": "A (Proxy) Minion-less Approach to Network Automation using Salt",
  "datePublished": "2019-06-17T08:10:00+00:00",
  "dateModified": "2019-07-17T08:10:00+00:00",
  "author": {
    "@type": "Person",
    "name": "Mircea Ulinic"
  },
   "publisher": {
    "@type": "Organization",
    "name": "Mircea Ulinic",
    "logo": {
      "@type": "ImageObject",
      "url": "https://mirceaulinic.net/img/avatar-icon.png"
    }
  },
  "description": "A (Proxy) Minion-less Approach to Network Automation using Salt"
}
</script>

Salt is currently one of the largely adopted automating frameworks, and perhaps
one of the most complete and flexible. But these - including support for
several architectures, large cardinal of native capabilities and features,
ability to easily extend it in your own environment, and so on - come with a
cost. The cost is setting up and maintaining an infrastructure to facilitate and
leverage these capabilities.

In the networking space, Salt has had native support for automating networks
beginning with release 2016.11.0 (codename Carbon) and these have been
continuously expanded since, now covering a considerable number of platforms -
either through [NAPALM](https://github.com/napalm-automation/napalm) (and the
available [community drivers](https://github.com/napalm-automation-community/)),
or various others natively integrated modules. Network gear is generally managed
through Proxy Minions, which is a derivative of regular Salt Minion allowing to
manage the targeted device remotely. Let's make one step back: as you may
know very well already, a regular Minion is a service managing the machine it
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

Most importantly, you don't need to learn how to use another framework, but the
same Salt methodologies that have been around for many years already, with the
experience of being run in thousands of environments and providing thousands of
features ready to use, out of the box.

``salt-sproxy``
---------------

salt-sproxy is an agentless approach to automation, while still using Salt, but
without requiring Proxy or regular Minions. It is a plugin, extending Salt, so
you can continue to benefit from the scalability, extensibility and flexibility
of Salt, while you don't have to manage thousands of (Proxy) Minion services.

I'll firstly present what it does and then we'll take a look at how to use. The
core idea is that ``salt-sproxy`` creates a subprocess for every device matching
the target executing the requested Salt function. Under each subprocess it's
running a lightweight version of the Proxy Minion - that is, the regular Proxy
Minion start up, including compiling the Pillar, collecting the Grains, but
without other components that wouldn't make sense in this case, i.e., Engines,
Beacons, Scheduler etc. In short, it pretty much only establishes the connection
to the remote network device, over the API of choice, then invokes the function
you requested; as soon as the execution of the Salt function is finished, it
closes the connection and terminates the subprocess.

``salt-sproxy`` can be very well installed on an existing Master, or even on
your personal computer, as it only requires one thing: be able to connect to the
network device from where you run it. Not only that you don't require any Proxy
Minions always running, but you don't actually need a Salt Master either. It's
as easy as that: install and use. It is [available on 
PyPI](https://pypi.org/project/salt-sproxy/) so you can just go ahead and
execute ``pip install salt-sproxy`` to install.

The usage is fairly simple, very similar to the syntax of the usual ``salt``
command:

```bash
$ salt-sproxy <target> <function> [<arguments>] [<options>]
```

Where:

- ``target`` is the target expression to select what devices to execute the
  command on. E.g., ``edge1.atlanta``, ``edge*.paris`` (to select all the edge
  routers in the Paris area), etc.
- ``function``: the Salt function to execute. There are several hundreds of Salt
  functions natively available. You can start exploring from
  [here](https://docs.saltstack.com/en/latest/ref/modules/all/).
- ``arguments``: arguments to be passed to the Salt function, if required. See
  the documentation for each function for usage details and examples.
- ``options``: options for the ``salt-sproxy`` command. See
  [here](https://salt-sproxy.readthedocs.io/en/latest/opts.html) the
  available options, or execute ``salt-sproxy --help`` on the CLI.

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
$ salt-sproxy edge1.thn.lon net.arp -c /path/to/salt/config/dir
```

With this command, it'll try to load the configuration from
``/path/to/salt/config/dir/master``.

The good news here is that (almost) everything is exactly preserved to what you
are already used to, and you can, e.g., use the
[Returner](https://docs.saltstack.com/en/latest/ref/returners/) interface to
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
post on
[Extending NAPALM's capabilities in the Salt environment](https://mirceaulinic.net/2019-04-24-extending-napalm-salt/):

```bash
$ salt-sproxy juniper-router example.version
```

To be able to load your custom modules, as explained in the blog post, you would
need to have the correct configuration for the ``file_roots``. The (positive)
difference is that you won't need to call ``saltutil.sync_modules`` before
calling your function, as everything is loaded on run time.

Quick start
-----------

Let's start with something very easy and use the ``dummy`` Proxy Minion to
connect to the local machine; we have the following configuration:

- The Master configuration file:

``/etc/salt/master``
```yaml
pillar_roots:
  base:
    - /etc/salt/pillar
```

- The Pillar Top file:

``/etc/salt/pillar/top.sls``
```yaml
base:
  minion1:
    - dummy
```

- The ``dummy`` Pillar:

``/etc/salt/pillar/dummy.sls``
```yaml
proxy:
  proxytype: dummy
```

To check that the Pillar is correctly setup, execute (you don't need a Master
running for this):

```bash
$ salt-run pillar.show_pillar minion1
proxy:
    ----------
    proxytype:
        dummy
```

With this configuration, you can go ahead and run:

```bash
$ salt-sproxy minion1 test.ping
minion1:
    True
```

As you can see, the dummy ``minion1`` is now usable without having to start up
a dedicated Proxy process for this. In the same way, you're now able to execute
any other Salt function, for example ``pip.list`` which would display the list
of the installed PIP-managed Python packages on the local computer (or wherever
you're executing salt-sproxy on).

If you prefer a visual of this example, take a look at the following capture:

<script id="asciicast-247697" src="https://asciinema.org/a/247697.js" async></script>

The same methodology can then be applied for connecting to network gear through
your Proxy module of choice. For example,
[``napalm``](https://docs.saltstack.com/en/latest/ref/proxy/all/salt.proxy.napalm.html);
update the Pillar and Top file accordingly:

``/etc/salt/pillar/top.sls``
```yaml
base:
  juniper-router:
    - junos
  arista-switch:
    - eos
```

``/etc/salt/pillar/junos.sls``
```yaml
proxy:
  proxytype: napalm
  driver: junos
  hostname: hostname-or-fqdn
  username: your-username
  password: your-password
```

Similarly it is a good idea to check using
``salt-run pillar.show_pillar juniper-router`` that the Pillar is indeed
correctly defined, then you're good to go, e.g.,

- Retrieve the ARP table of ``juniper-router``:

```bash
$ salt-sproxy juniper-router net.arp
juniper-router:
    ----------
    comment:
    out:
        |_
          ----------
          age:
              849.0
          interface:
              fxp0.0
          ip:
              10.96.0.1
          mac:
              92:99:00:0A:00:00
        |_
          ----------
          age:
              973.0
          interface:
              fxp0.0
          ip:
              10.96.0.13
          mac:
              92:99:00:0A:00:00
        |_
          ----------
          age:
              738.0
          interface:
              em1.0
          ip:
              128.0.0.16
          mac:
              02:42:AC:13:00:02
    result:
        True
```

- Load a configuration change:

```bash
$ salt-sproxy juniper-router net.load_config text='set system ntp server 10.10.1.1'
juniper-router:
    ----------
    already_configured:
        False
    comment:
    diff:
        [edit system]
        +   ntp {
        +       server 10.10.1.1;
        +   }
    loaded_config:
    result:
        True
```

I have prepared a CLI capture for this example as well, which you can watch
below:

<script id="asciicast-247726" src="https://asciinema.org/a/247726.js" async></script>

As promised, the methodology remains the same, without the headache of managing
thousands of always running processes.

Does it work only with NAPALM?
------------------------------

No. You can use any of the available
[Proxy modules](https://docs.saltstack.com/en/latest/ref/proxy/all/index.html)
in the exact same way. The ``salt-sproxy`` relies on the Proxy modules to
initiate the connection, but it doesn't require the Proxy process to be running.
If you have a device that isn't currently supported by Salt natively, just write
a Proxy module for it - see
[this guide](https://docs.saltstack.com/en/latest/topics/proxyminion/index.html#proxymodules):
put the module into the ``salt://_proxy`` directory, and then make sure that the
module is referenced in the ``proxytype:`` Pillar field. If you're unsure what
``salt://_proxy`` means, please check
[this](http://mirceaulinic.net/2019-04-24-extending-napalm-salt/) article again.

Suppose we write a custom Proxy module, e.g., ``customproxy.py`` which we put
under ``/srv/salt/extmods/_proxy``. Then, to use it, when targeting a device,
e.g., ``some-device``, in the Pillar you'll need to have the following
structure:

```yaml
proxy:
  proxytype: customproxy
  ~~~ connection credentials ~~~
```

To check that the data is indeed available for the ``some-device`` Minion,
you can always run the following command:

```bash
$ salt-run pillar.show_pillar some-device
```

Then you should be all set to give it a go:

```bash
$ salt-sproxy some-device test.ping
```

I am not going to detail on this further, but you can follow the notes from
[this guide](https://docs.saltstack.com/en/latest/topics/proxyminion/index.html)
with the minor differences for salt-sproxy described above.

In the exact same way, you can use any of the [natively available Proxy 
modules](https://docs.saltstack.com/en/develop/ref/proxy/all/index.html) and
manage your [VMWare 
ESXi](https://docs.saltstack.com/en/develop/ref/proxy/all/salt.proxy.esxi.html#module-salt.proxy.esxi),
[Chronos cluster](https://docs.saltstack.com/en/develop/ref/proxy/all/salt.proxy.chronos.html#module-salt.proxy.chronos),
[Bluecoat SSL Decryption 
devices](https://docs.saltstack.com/en/develop/ref/proxy/all/salt.proxy.bluecoat_sslv.html#module-salt.proxy.bluecoat_sslv),
and many others.

So what's the catch?
--------------------

Does it sound too good to be true? Well, it is that good, and there isn't any
catch. It might not be immediately obvious what are the implications and the
benefits to using this methodology, but it made me very happy when I've been
able to put this together.

There are some differences however, that you should be aware of. The usual
``salt`` command when executing, spreads out a job to all the connected Minions,
and those that match the target are going to reply. As ``salt-sproxy``, by
design, doesn't have any Minions connected (as they are not running), it won't
be aware of what Minions should match your target. For this reasoning, it needs
some "help". Salt already has had a subsystem named Salt SSH which works in a
similar way, i.e., manage a remote system without having a Minion process up and
running, connecting to the device over SSH. If interested, you can read more
about Salt SSH [here](https://docs.saltstack.com/en/latest/topics/ssh/).

Due to this similarity, the Salt SSH system has the same limitation which made
me think: "then why not have the same solution?". So I borrowed the
[``Roster``](https://docs.saltstack.com/en/latest/topics/ssh/roster.html)
interface from Salt SSH, which is another pluggable Salt interface, which
provides a list of devices and their connection credentials, given a specific
target. In order words, you sometimes might have more complex targets that a
single device or a list (e.g., you can have a regular expression --
``edge{1,2}.thn.*``) and so on; in that case, you'd need a Roster. There are
several Roster modules available natively, and you can explore them
[here](https://docs.saltstack.com/en/latest/ref/roster/all/index.html#all-salt-roster).

At the end of the day, it is the same methodology with other well
know automation tools such as Ansible where you have to provide an *inventory*
with all the devices it should be aware of.
I will provide an usage example in the following paragraph, using the Ansible
Roster module.

Migrating from Ansible to Salt and salt-sproxy
----------------------------------------------

I've seen a lot Ansible users that were interested to migrate to Salt, however
the radically different mentality that you need to adopt (including the always
running processes) was a blocker for many. Hopefully this should no longer be
the case anymore. If you're already an Ansible user, ``salt-sproxy`` should
make it even easier to migrate to Salt.

As briefly presented above, to be able to match more sophisticated targets 
(groups of devices), you may need a Roster. Even easier for you probably to
provide a Roster file, as you might already have an Ansible inventory file.
Simply move/copy your Ansible inventory to ``/etc/salt/roster``, and tell
``salt-proxy`` to use it by configuring the ``proxy_roster`` option:

``/etc/salt/master``
```yaml
proxy_roster: ansible
```

Should you prefer to store your Ansible inventory under a different path, you
can use the ``roster_file`` option, e.g.,

``/etc/salt/master``
```yaml
proxy_roster: ansible
roster_file: /path/to/ansible/inventory
```

One particular difference to always remember is that the Ansible Roster /
inventory file doesn't have to provide the connection details, as those are
already managed into the Pillar, as detailed previously.

If the configuration is correct, you can run the following to verify that the
inventory is interpreted properly - the command should display the entire list
of devices salt-sproxy should be aware of:

```bash
$ salt-sproxy '*' --preview-target
```

When the configuration is correctly setup, you should be able to check the list
of devices matched by your target expression (determined via the Roster
interface):

```bash
$ salt-sproxy <tgt> --preview-target
- edge1.seattle
- edge1.vancouver
- edge1.atlanta
- edge2.atlanta
- edge1.raleigh
- edge1.la
- edge1.sfo
```

Or a specific group, as defined in the inventory file, e.g.,

```bash
$ salt-sproxy -N southeast --preview-target
- edge1.atlanta
- edge2.atlanta
- edge1.raleigh
```

More extensive details are documented at:
https://salt-sproxy.readthedocs.io/en/latest/roster.html,
and configuration examples at
https://github.com/mirceaulinic/salt-sproxy/tree/master/examples. For Ansible
specifically, you might want to focus on these two sections:
https://salt-sproxy.readthedocs.io/en/latest/roster.html#roster-example-ansible
and
https://github.com/mirceaulinic/salt-sproxy/tree/master/examples/ansible/.

In this way, you can benefit from Ansible's simplicity and Salt's power and
flexibility (and ease of extensibility) altogether.

Migrating from Proxy Minions to salt-sproxy
-------------------------------------------

I do not encourage turning off all your Proxies and replacing them with
salt-sproxy, however it might make sense to tear down some of them. It probably
boils down to how many operations per second you are aiming for. Either way,
salt-sproxy can work very well even with devices whose Proxy is already up and
running, without any issues. Assuming that your Proxy is properly configured,
you should be able to go ahead and execute arbitrary commands immediately, e.g.,

```bash
$ salt-sproxy 'iosxr-router' route.show '0.0.0.0/0'
iosxr-router:
    # TODO get output from an XR
```

The previous command would show the routes to ``0.0.0.0/0`` on the
``iosxr-router`` which is supposed to be already configured as a Proxy.
Confirming that works well, you can go ahead and turn off your Proxy service
for ``iosxr-router``, and so on.

Why not use both Proxy Minions and salt-sproxy
----------------------------------------------

You can do that too. ``salt-sproxy`` has been designed in such a way that if you
already have Proxy Minions up and running, you can start using it straight away -
as seen in the previous paragraph. If you decide not to turn them off, the CLI
option ``--use-existing-proxy`` is going to execute the command on the existing
Proxy Minions, whenever possible. In case your target is valid and didn't match
an existing Proxy Minion, then ``salt-sproxy`` will detect this and will try to
execute the Salt function locally.

If you will want to have this as the default behaviour, you can set the
following option in the Master config:

```yaml
use_existing_proxy: true
```

You can find out more about running mixed environments at
https://salt-sproxy.readthedocs.io/en/latest/mixed_envs.html

Even-driven automation? Not a problem
-------------------------------------

Have you raised eyebrows reading this title? If you're a seasoned Salt
user, you might be wondering what does a program that is executed on demand,
have anything to do with the event-driven paradigm? Well, you are still going
to need a Master running. But now the good news: the master is all you need,
which I think is a sufficiently good trade-off to have only one process always
running that practically allows you to leverage the entire power of Salt.

The recipe is fairly simple, I see three directions:

1. Not interested in event-driven automation - use just ``salt-sproxy``.
2. Interested in event-driven automation, but not in a highly dynamic
  environment - use ``salt-sproxy`` together with a Salt Master.
3. Interested in event-driven automation, in a highly dynamic environment - use
  Proxy Minions.

If you're still here, it means you're probably curious about (2). As I mentioned
already, you are going to need a Salt Master up and running. Using the CLI
option ``--events``, ``salt-sproxy`` is able to inject events on the Salt bus,
which you can then trigger reactions in response to these events, or - why not -
export these events as a way to monitor who is executing what. That, for example,
is a nice and easy win for compliance and ensuring you have visibility in your
network. For example, executing one of the previously used commands, with the
``--events`` option:

```bash
$ salt-sproxy minion1 test.ping --events
minion1:
    True
```

Watching the event bus in a separate terminal while running the previous
command:

```bash
$ salt-run state.event pretty=True

20190604102025230045	{
    "_stamp": "2019-06-04T09:20:25.230783",
    "minions": [
        "minion1"
    ]
}
proxy/runner/20190604102025232023/new	{
    "_stamp": "2019-06-04T09:20:25.232688",
    "arg": [],
    "fun": "test.ping",
    "jid": "20190604102025232023",
    "minions": [
        "minion1"
    ],
    "tgt": "minion1",
    "tgt_type": "glob",
    "user": "mircea"
}
proxy/runner/20190604102025232023/ret/minion1	{
    "_stamp": "2019-06-04T09:20:25.330420",
    "fun": "test.ping",
    "fun_args": [],
    "id": "minion1",
    "jid": "20190604102025232023",
    "return": true,
    "success": true
}
```

The events pushed on the Salt bus have the same structure as the native Salt
execution events, the only difference being that the tag is registered under the
``proxy/runner`` namespace vs. ``salt/job`` as the usual Salt events.
See https://salt-sproxy.readthedocs.io/en/latest/events.html for further details
and examples.

If you would like to have Salt events by default, simply add this line to
your Master config:

```yaml
events: true
```

With this, enabling an Engine, for instance the
[``http_logstash``](https://docs.saltstack.com/en/latest/ref/engines/all/salt.engines.http_logstash.html#module-salt.engines.http_logstash)
Engine, on the Master (check out the
[documentation](https://docs.saltstack.com/en/latest/ref/engines/all/salt.engines.http_logstash.html#module-salt.engines.http_logstash)
how to do so), Salt is going to export the selected (or all) events into
[Logstash](https://www.elastic.co/products/logstash) which instantly gives you
visibility on who, what and when applies a configuration change, or any other
command touching your network. Frankly speaking, I don't think it can't be any
easier than that. :-)

The same goes with any other Salt components watching the event bus: you can
forward your events into a database using a Returner, forward execution errors
to [Sentry](https://sentry.io/welcome/) using
[this](https://docs.saltstack.com/en/latest/ref/returners/all/salt.returners.sentry_return.html)
Returner, or even send
[SMSs](https://docs.saltstack.com/en/latest/ref/returners/all/salt.returners.sms_return.html)
in case of critical events. In all these cases, everything you have to do is
providing the configuration as documented. Take a look at the available
[Returners](https://docs.saltstack.com/en/latest/ref/returners/index.html#full-list-of-returners)
and [Engines](https://docs.saltstack.com/en/latest/ref/engines/all/index.html).
As usually, there are many interfaces natively available, but it is possible
that they can't cover your needs exactly; in that case, it's very easy to write
a Returner or an Engine in your own environment.

But wait: there's more: it also works the other way around; you can trigger
jobs to be executed against network devices, in response to various events. The
actual core of ``salt-sproxy`` is a Salt Runner named ``proxy``, which is loaded
dynamically on run time. With this Runner, you can execute Salt commands on
devices, without having Minions.

Let's take an example from one of my previous posts,
[Event-driven network automation using napalm-logs and 
Salt](https://mirceaulinic.net/2017-10-19-event-driven-network-automation).
Take a moment and read, or re-read that post as it's essential in order to 
fully understand the following.

One of the Reactor files I exemplified was the following (triggered in response
to an
[interface down event](http://napalm-logs.com/en/latest/messages/INTERFACE_DOWN.html)
from [napalm-logs](https://github.com/napalm-automation/napalm-logs)):

``/etc/salt/reactor/if_down_shutdown.sls``
{% raw %}
```yaml
shutdown_interface:
  local.net.load_template:
    - tgt: {{ data.host }}
    - kwarg:
        template_name: salt://templates/shut_interface.jinja
        interface_name: {{ data.yang_message.interfaces.interface.keys()[0] }}
```
{% endraw %}

This Reactor file can be transformed in the following way, to use execute the
commands through the ``proxy`` Runner:

``/etc/salt/reactor/if_down_shutdown.sls``
{% raw %}
```yaml
shutdown_interface:
  runner.proxy.execute:
    - tgt: {{ data.host }}
    - function: net.load_template
    - kwarg:
        template_name: salt://templates/shut_interface.jinja
        interface_name: {{ data.yang_message.interfaces.interface.keys()[0] }}
```
{% endraw %}

As you can notice, the difference is small: ``local.net.load_template`` becomes
``runner.proxy.execute_devices`` which tells the Reactor to invoke the
``proxy.execute_devices`` Runner, which in turn executes the ``net.load_template``
Salt function against the device(s) from the ``tgt`` key. The arguments under
``kwargs`` are preserved, with the same effect. The difference is however in the
way it works: while previously with ``local.net.load_template`` the execution is
sent to the Minion managing the device, changing to
``runner.proxy.execute_devices`` the execution takes place locally. This is
practically the main difference between using Proxy Minions and salt-sproxy in
this context.

I could expand longer on this, but I'll probably follow up with a dedicated post
if this is not clear and elaborate as much as possible. For now, I hope this
brief introduction is intriguing - at least. :-)

Execution over the Salt REST API? Sorted!
-----------------------------------------

Through the same Runner as previously, after enabling the Salt HTTP API (see
https://docs.saltstack.com/en/latest/ref/netapi/all/salt.netapi.rest_cherrypy.html
to learn how to correctly set it up), you should be able to execute, e.g.,

```bash
$ curl -sS localhost:8080/run -H 'Accept: application/x-yaml' \
  -d client='runner' \
  -d fun='proxy.execute' \
  -d username='mircea' \
  -d password='pass' \
  -d eauth='pam' \
  -d tgt='minion1' \
  -d function='test.ping' \
  -d sync='true'
return:
- minion1: true
```

Or from Python:

```python
>>> resp = session.post('http://localhost:8080/run', json={
... 'client': 'runner',
... 'fun': 'proxy.execute',
... 'username': 'mircea',
... 'password': 'pass',
... 'eauth': 'pam',
... 'tgt': 'minion1',
... 'function': 'test.ping',
... 'sync': True})
>>> resp.json()
{u'return': [{u'minion1': True}]}
```

The examples above are the equivalent of the CLI
``salt-sproxy minion1 test.ping`` used in the previous examples.

See https://salt-sproxy.readthedocs.io/en/latest/salt_api.html for more details
and usage examples.

Does it work on Windows?
------------------------

I don't know. I haven't tried, as I don't have a Windows machine available, and,
to be fair, I have no idea how to use Python on Windows. But please go ahead and
give it a try, then let me know. Normally, if you're able to install ``salt``
from PyPI, ``salt-sproxy`` should be usable too - that is my guess, at least.

In the [``salt-sproxy``](https://github.com/mirceaulinic/salt-sproxy) repository
I've added a script that should normally facilitate the installation on Unix
machines: https://raw.githubusercontent.com/mirceaulinic/salt-sproxy/master/install.sh
(and usage details at https://salt-sproxy.readthedocs.io/en/latest/install.html).

If you can, please submit a PR to provide a ``.bat`` script or
anything that provides the equivalent functionality on Windows - the community
would greatly appreciate that. Thanks in advance!

Conclusions
-----------

I am super excited to release this project, and I hope it is going to
help a lot. Salt is a beautiful tool, but often overlooked due to its high entry
barrier and loads of requirements. I believe that ``salt-sproxy`` is going to
make it *much* easier for everyone to start automating, while allowing you to
enjoy the most beautiful parts in Salt, together with its flexibility,
extensibility and tons of features ready to be used, including the REST API and
the event-driver methodology. I recommend you to take a look at the
[Quick Start](https://salt-sproxy.readthedocs.io/en/latest/#quick-start) section
of the documentation and convince yourself.

I don't think the ``salt-sproxy`` is meant to replace the existing Proxy Minion-
based approach, but rather fill in some gaps where the Proxy Minions fall short,
or are a burden to manage. As I mentioned in the beginning of this post, in my
opinion, Proxy Minions would always make sense in dynamic environments, or
to manage devices susceptible to frequent change. On the long term, I would
assume that the winning combination is running both Proxy Minions and
``salt-sproxy`` concomitantly, although there may be exceptions. It depends very
much on your operations, the network topology, what are your goals and many
other aspects on where to draw the line; I think that the rule of thumb here is
to evaluate which devices require frequent changes / interaction (for which
you would start Proxy processes), and which are more statical (which you'd
probably manage using the ``salt-sproxy``).

If you also like the project, remember to star it on GitHub:
https://github.com/mirceaulinic/salt-sproxy and why not Tweet about it, and let
your friends know. The larger the community, the easier is going to be for every
one of us!

I am looking forward to hearing your feedback. I would love to make it easier 
for everyone to get started to automate, so please help improve the docs or add
your usage examples to
https://github.com/mirceaulinic/salt-sproxy/tree/master/examples so other
engineers can follow them.
