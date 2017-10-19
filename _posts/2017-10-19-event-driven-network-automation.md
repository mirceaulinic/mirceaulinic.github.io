---
layout: post
title: Event-driven network automation
subtitle: Using napalm-logs and Salt
bigimg: /img/engine.jpg
---

One of the most common misconceptions is that network automation is *only* about
configuration management. While this is indeed a very important milestone, it
is not the ultimate achievement you want for your network. We inherit most of
the methodologies from the server side, and I can tell that the network
environments are generally much more dynamic: there are *many* internal and
external events that we are aware of, but we still continue doing things
*manually*. And by manually I mean any sort of command that you execute on the
CLI - either directly on the device, or via a configuration management tool.
Make no mistake: while having a configuration management tool ensures
consistency and brings plenty of benefits, but considerable percentage of the
tasks you execute daily (or hourly) can be *fully automated*, via event-driven
network automation. With this, I don't refer only to configuration changes, but
also the boring email notifications you need to type manually when a BGP
neighbor has been down for several days, an interface is flapping, or checking
with the fiber provider why the optics level is still under the threshold: all
of these can be easily automated.

Another frequent misconception is that Salt is (just) a configuration management
tool. It is that too, but among others (including cloud provisioning,
even reactors etc.), Salt is also an event-driven automation and orchestration
framework - and, in my opinion, it is the most complete.

Sources of events
-----------------

There are several sources of information in every network, some have been there
for years, but we didn't exploit their entire potential just yet.

# SNMP Traps

The SNMP traps represent one of the most commonly used source of events. Just
that they have been mainly used as notifications, while they can be used very
well to feed an event-driven framework, and trigger completely automated actions.
Unfortunately, we can still identify the following pattern: a notification comes
in, then the network engineer either applies manually a configuration change on
the CLI of the network device, or - more recently - executes a command (from a
server or the local computer) that pushes the desired configuration; then,
eventually writes an email, and it's the same email every single time, only the
interface name being different. I hope you see what's wrong here: this entire
process can be fully automated as well.

# Syslog

Syslog messages have been as well present for so many years - that's no news!
While the SNMP traps are used, at least, as notifications, syslog messages go
very often directly into the bin. Please take a moment and ask yourself three
questions: 1) Do you store the syslog messages anywhere? 2) If you do, do you
know where is that server and how to retrieve the messages? 3) Do you use them
anyhow (excluding TAC cases)?

There's plenty of useful information in these syslog messages: they can tell
you that an interface is flapping, the optics level is below the threshold,
your device is not NTP synchronised anymore, chassis alarms, or when one of
your BGP neighbors is leaking their entire routing table to you, and many
others that can be critical for your infrastructure.

# Streaming Telemetry

Streaming telemetry is the new black; as in opposite to SNMP which requires you
to fetch data every so often, the device sends notifications when it has
anything to communicate. Besides this methodology that is more efficient, you
don't receive snippets of text, but rather structured documents. The structure
of these documents is standardised as well, in the
[OpenConfig](http://www.openconfig.net/) and IETF YANG
models. The main drawback currently is that vendors such as Juniper or Cisco
support it only on very new software distributions, e.g., you need at least
Junos 15.1 (as usually, depending on the platform), or IOS-XR 6.1.1 or higher.
I am confident this is going to get better in time however!

Gabriele Gerbino wrote a very nice [blog post on Streaming Telemetry](projectme10.wordpress.com/2016/11/14/telemetry-streaming-on-cisco-ios-xr-and-nxos/)
that I recommend you to read.

Using napalm-logs for syslog messages
-------------------------------------

Syslog messages are simple chunks of text that represent a notification, having
different levels of severity. But here arises the eternal problem of
the networking community: the structure of the syslog messages is not
cross-platform.

Below is a syslog message from a Junos device notifying that an interface
is down:

```text
<28>Jul 20 21:45:59 edge01.bjm01 mib2d[2424]: SNMP_TRAP_LINK_DOWN: ifIndex 502,
ifAdminStatus down(2), ifOperStatus down(2), ifName xe-0/0/0
```

The same notification from a Cisco IOS-XR would be:

```text
<187>94307: gw2.acy1 LC/0/2/CPU0:Jul  7 20:16:14.834 : ifmgr[214]:
%PKT_INFRA-LINK-3-UPDOWN : Interface TenGigE0/2/0/4, changed state to Down
```

This is why my colleague Luke Overend and I started working on a library to
normalise the syslog messages, in a vendor-agnostic way. And we soon learnt that
the most suitable structures have been already defined in the OpenConfig and
IETF YANG models.

I have elaborated more on this topic in a
[NAPALM Automation blog post](https://napalm-automation.net/napalm-logs-released/).

The syslog messages can be directed straight from the device to the napalm-logs
process via UDP, or TCP, or via brokers or other systems such as Apache Kafka,
ZeroMQ etc. The resulting structured data is binary serialised, encrypted and
signed before being published over various channels including ZeroMQ, Kafka,
or RabbitMQ.

For example, napalm-logs would map the text syslog messages above into the
[openconfig-interfaces](ops.openconfig.net/branches/master/openconfig-interfaces.html)
YANG model:

```json
{
  "error": "INTERFACE_DOWN",
  "facility": 23,
  "host": "gw2.acy1",
  "ip": "5.6.7.8",
  "os": "iosxr",
  "severity": 3,
  "timestamp": 1499458574,
  "yang_message": {
      "interfaces": {
          "interface": {
              "TenGigE0/2/0/4": {
                  "state": {
                      "oper_status": "DOWN"
                  }
              }
          }
      }
  },
  "yang_model": "openconfig-interfaces"
}
```

And the structure is exactly the same when receiving the notification from
a Junos device, IOS-XR or any other platform!

One of the most important details to remember is that napalm-logs continuously
listens to syslog messages as test, and continuously publishing structured
documents, where multiple clients can connect. The clients can be event-driven
frameworks such as Salt or StackStorm, or simple scripts written in any
programming language you prefer.

Importing syslog notifications into the Salt event bus
------------------------------------------------------

As mentioned in the previous paragraph, Salt can be a client for napalm-logs,
importing the structured documents into the event bus. In the Nitrogen release
(2017.7), we have integrated an Engine,
[napalm-syslog](https://docs.saltstack.com/en/latest/ref/engines/all/salt.engines.napalm_syslog.html),
for this exact purpose. It is very easy to configure, you mainly need to tell
what is the IP address and port where Salt should be listening to the messages
from napalm-logs. You can read more details under
[*The napalm-syslog Salt engine*](https://napalm-automation.net/napalm-logs-released/)
from the same NAPALM Automation blog post.

An important detail to note here is that the messages imported into the Salt
bus have a *tag*; the associated tag for the previous example is
``napalm/syslog/iosxr/INTERFACE_DOWN/gw2.acy1`` (*each tag delimiting a
namespace*), the complete event structure being:

```json
napalm/syslog/iosxr/INTERFACE_DOWN/gw2.acy1 {
  "error": "INTERFACE_DOWN",
  "facility": 23,
  "host": "gw2.acy1",
  "ip": "5.6.7.8",
  "os": "iosxr",
  "severity": 3,
  "timestamp": 1499458574,
  "yang_message": {
      "interfaces": {
          "interface": {
              "TenGigE0/2/0/4": {
                  "state": {
                      "oper_status": "DOWN"
                  }
              }
          }
      }
  },
  "yang_model": "openconfig-interfaces"
}
```

The body of the event is called *data*.

Fully automated jobs
--------------------

Once everything is up and running, your network is ready to receive the
event-driven automation blessings :-)

As every event on the Salt bus has a *tag*, it is very easy to identify events
using the [Reactor system](https://docs.saltstack.com/en/latest/topics/reactor/)
(please do read the Reactor documentation).

Let's consider the following reactor configuration:

```yaml
reactor:
  - 'napalm/syslog/*/INTERFACE_DOWN/*':
    - salt://reactor/if_down_shutdown.sls
    - salt://reactor/if_down_send_mail.sls
```

The reactor tries to match the event tags against the
``napalm/syslog/*/INTERFACE_DOWN/*`` expression; under each *namespace*, we can
have either the exact string expected, or shell globbing. In this case, for
instance, ``*`` will match anything: the first one corresponds to the network
OS namespace, while the second one will match any hostname. This expression
will therefore match the ``napalm/syslog/iosxr/INTERFACE_DOWN/gw2.acy1`` tag
from the above, as well as ``napalm/syslog/junos/INTERFACE_DOWN/edge01.lyc01``,
or ``napalm/syslog/nxos/INTERFACE_DOWN/sw01.par01``; but it will not match
``napalm/syslog/iosxr/NTP_SERVER_UNREACHABLE/gw2.acy1``, for example.

When an event is matched, the reactor will invoke one more reactor SLS
descriptors. (I remind you that SLS is Jinja+YAML, by default, but extensible
to any combination of
[renderers](https://docs.saltstack.com/en/latest/ref/renderers/))

Inside these SLS files, we describe what Salt should do when the event occurs.
In this example, when there's an interface down notification, there will be
triggered two automated actions: 1) Administratively shutdown the interface
(``salt://reactor/if_down_shutdown.sls``) & 2) Send out an email
(``salt://reactor/if_down_send_mail.sls``). Both files are stored under the
Salt filesystem (hence the ``salt://``), so they are physically found under
one of the [``file_roots``](https://docs.saltstack.com/en/latest/ref/file_server/file_roots.html)
paths, e.g.,:

```yaml
file_roots:
  base:
    - /etc/salt
```

With this configuration, the file ``salt://reactor/if_down_shutdown.sls`` has
the absolute path ``/etc/salt/reactor/if_down_shutdown.sls``,
``salt://reactor/if_down_send_mail.sls`` is
``/etc/salt/reactor/if_down_send_mail.sls``, etc. The great advantage of using
the ``salt://`` URI is that you can move the entire environment without any
configuration change (except ``file_roots`` on the Master).

Let's have a look at each of these SLS files:

``/etc/salt/reactor/if_down_shutdown.sls``
```yaml
shutdown_interface:
  local.net.load_template:
    - tgt: {{ data.host }}
    - kwarg:
        template_name: salt://templates/shut_interface.jinja
        interface_name: {{ data.yang_message.interfaces.interface.keys()[0] }}
```

- ``shutdown_interface`` is just a human understandable name (it can be anything);

- ``local`` is the
[reactor type](https://docs.saltstack.com/en/latest/topics/reactor/#types-of-reactions)
that will tell Salt to invoke an execution function --
[``net.load_template``](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_network.html#salt.modules.napalm_network.load_template) in this case.

- ``data`` is the event *data*, the body of the event, so ``data.host`` will give
the information from the ``host`` field of the message.

When interpreting the Reactor SLS, Salt will firstly render the Jinja, e.g.,
when the field ``host`` is ``gw2.acy1``, the line ``- tgt: {{ data.host }}``
becomes ``- tgt: gw2.acy1``, which tells Salt to execute ``net.load_template``
on the Minion having the ID ``gw2.acy1``. (This assumes the Minion ID is the
host of the device, but the logic can be as complex as suitable for the business
case). Under ``kwarg``, we specify the arguments used to execute
``net.load_template``: ``template_name`` specifies the template source, while
``interface_name`` is the variable sent to the
``salt://templates/shut_interface.jinja`` Jinja template. To make this easier to
understand, here's the resulting YAML after Salt renders the SLS using Jinja:

```yaml
shutdown_interface:
  local.net.load_template:
    - tgt: gw2.acy1
    - kwarg:
        template_name: salt://templates/shut_interface.jinja
        interface_name: TenGigE0/2/0/4
```

Basically, the CLI equivalent of this reactor SLS is:
``$ sudo salt 'gw2.acy1' net.load_template salt://templates/shut_interface.jinja interface_name=TenGigE0/2/0/4``.

For completeness, I will provide a very minimalist example for the
``salt://templates/shut_interface.jinja`` template. Similarly, the template
can be specified using absolute path, or from the Salt filesystem via ``salt://``,
or using one of the following URI selectors: ``http://`` / ``https://``,
``ftp://``, ``s3://``, ``swift://``.

``/etc/salt/templates/shut_interface.jinja``

{% raw %}
```jinja
{%- if grains.os == 'iosxr' %}
interface {{ interface_name }}
  shutdown
{%- elif grains.os == 'junos' %}
deactivate interface {{ interface_name }};
{%- endif %}
```
{% endraw %}

The variable ``interface_name`` is sent to the template by the reactor SLS (see
under ``kwarg``).

You can learn how to write smart, cross-vendor Jinja templates reading
[this tutorial](https://docs.saltstack.com/en/develop/topics/network_automation/index.html#napalm), or from the
[Network Programmability and Automation](http://shop.oreilly.com/product/0636920042082.do),
where I have been invited to write a chapter about Salt. Similarly, I expanded
a little on templating configuration management in Salt, in a
[previous post](https://mirceaulinic.net/2016-12-15-configuration-management/).

The second Reactor SLS invoked when an interface down notification occurs is
``salt://reactor/if_down_send_mail.sls``:

``/etc/salt/reactor/if_down_send_mail.sls``

{% raw %}
```yaml
{%- set if_name = data.yang_message.interfaces.interface.keys()[0] %}

send_email:
  local.smtp.send_msg:
    - tgt: {{ data.host }}
    - arg:
        - mircea@example.com
        - Interface down notification email body.
    - kwarg:
        subject: Interface {{ if_name }} is down
```
{% endraw %}

For a better understanding of the Reactor System, I highly recommend reading
[this document](https://docs.saltstack.com/en/latest/topics/reactor/) thoroughly.

After configuring these files, we can watch how Salt automatically disables the
interface, and sends out an email.

This is already a pretty ambitious goal, but, as one can easily notice, we can
achieve this very easily, in about 20-30 lines of configuration. But, as usually,
you need to know where and what to configure, which becomes a matter of reading
lots of good documentation.

What about SNMP Traps and Streaming Telemetry
------------------------------------------------

Streaming telemetry is in general straightforward and we only need to import
the data into the Salt bus. SNMP traps are generally collected into an
alert manager, e.g., [Prometheus](https://prometheus.io/), which, in general
support webhooks that can be used to inject events into the Salt bus. I will
expand on these two topics later.

Conclusions
-----------

Reading this post, I want you to remember that network automation is not just
about configuration management; every single boring task of your daily job can
be automated. My message is: there are several ways your network is trying to
communicate with you, and it sends you millions of messages every hour; don't
ignore it!

Using napalm-logs you can process and automate many internal notifications,
in a uniform and vendor agnostic way. Injecting them into the Salt event bus,
you can very easily implement the event-driven automation in your network!
