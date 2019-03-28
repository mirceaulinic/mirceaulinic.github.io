---
layout: post
title: Extending NAPALM's capabilities in the Salt environment
---

Having the privilege to maintain one of the widely used tools for network
automation, NAPALM, I am ofen asked various questions, many of them related to
the extensibility of the library. It is a given, and a design choice that NAPALM
does not and cannot possibly fulfill all the existing scenarios and features
when interacting with the network device - either retrieving or pushing data.

Some of the requests we receive do make sense to be implemented into the NAPALM
core library, however we are also seeing a large number that would solve a very
particular need, or subset. I am also one of the people that are using NAPALM
and had this sort of need to implement a niche requirement or a very specific
business logic that would definitely not have its place into the public library.

At the beginning, I started extending NAPALM into a custom library, by
[subclassing](https://www.digitalocean.com/community/tutorials/understanding-class-inheritance-in-python-3)
NAPALM's native classes (i.e., JunOSDriver, EOSDriver, etc.) and
extending them by adding more methods I needed. While this approach was easy
enough, after writing the code, there were two additional steps required: 1.
release a new version of this custom library, and 2. install it. While both
are easy enough in theory, when releasing code often (several times a day)
it can get tedious, especially when you have to mess with Python's poor
packaging system, and, on top of that, it is little point to do so when using
Salt which provides mature ways of spreading out new code, out of the box. Salt's
native filesystem can be used to release new code in the form of Execution
Modules. In the latest Salt release, 2019.2.0 - codename Fluorine, there are
many new features that expose the low level API that NAPALM is using (and beyond
that), which you can invoke from your own custom modules to implement the
business logic you need. The implementation is much easier that it sounds.
Check out the [release notes](https://docs.saltstack.com/en/develop/topics/releases/2019.2.0.html#napalm)
to before going any further.

*Note*: If you are not running 2019.2.0 yet, dont' worry, you are covered: all
you have to do in order to follow when I'm describing in the steps below, is to
copy the files directly from GitHub. For example, one of the files required is
[napalm_mod.py](https://github.com/saltstack/salt/blob/2019.2/salt/modules/napalm_mod.py).
To be able to use this module with older Salt releases, simply copy this file
into one of your ``file_roots`` directory. For example, if one of the
``file_roots`` directories is ``/etc/salt/extmods``, then you only have to
execute, e.g.,

```bash
$ cd /etc/salt/extmods
$ mkdir _modules && cd _modules/
$ wget https://raw.githubusercontent.com/saltstack/salt/2019.2/salt/modules/napalm_mod.py
```

*Remember* to always download the file using the _raw_ link from GitHub,
otherwise you'll download an HTML document instead of a Python module. :-)

The same goes with any other module type to port features from 2019.2.0 and use
them in older releases. Check out the list of features for network automation
available wih this release: [Salt 2019.2.0 release 
notes](https://docs.saltstack.com/en/develop/topics/releases/2019.2.0.html).

To make it easier, I have added them in the [napalm-salt](https://github.com/napalm-automation/napalm-salt)
GitHub repository, under the [``fluorine`` 
directory](https://github.com/napalm-automation/napalm-salt/tree/master/fluorine),
so you can simply clone the repo and add the path to your ``file_roots``.

Additionally, if you want to quickly have a play with these new features, you
might want to take a look at the [``napalmautomation/salt-master`` Docker 
image](https://hub.docker.com/r/napalmautomation/salt-master/tags), currently
offering two versions: ``2018.3.3-fluorine`` providing the ports for the
Fluorine release (2019.2.0), based on the 2018.3.3 Salt stable code, and
``2017.7.8-fluorine`` based on the 2017.7.8 stable release, respectively. For
example, if you want to run the 2017.7.8 release armed with the modules
available in 2019.2.0, you'd execute, e.g.,
``$ docker run -it napalmautomation/salt-master:2017.7.8-fluorine``.

These features in the latest Salt release provide low level interaction with the
network device, by exposing you directly the available API (if any, otherwise
you can use screen scraping via the Netmiko library). Using these, you can very
easily extend Salt's capabilities for your own needs, but writing a small
Execution Module. That said, let's take a look at how to write Execution Modules
in your own environment.

Writing Execution Modules is easy
=================================

I borrwoed this title from the official Salt documentation for [writing
Execution Modules](https://docs.saltstack.com/en/latest/ref/modules/). This
document explains very well this subject, and covers many aspects that you may
need sooner or later. Please take a moment and skim through it, then re-read it
thoroughly later. I wouldn't have much to add to it, however I would like to
insist a little on the aspect of the ``file_roots`` which is often neglected or
incompletely understood.

I will start by linking to the [Salt file server](https://docs.saltstack.com/en/latest/ref/file_server/)
documentation, and I will speak more specific in the context of Execution
Modules. Say you have the following configuration on the Master (this is what I
prefer to use):

```yaml
file_roots:
  - /etc/salt
  - /etc/salt/states
```

Salt is going to load your custom Execution Modules from both ``/etc/salt/_modules``
and ``/etc/salt/states/_modules`` when available. To keep it clear, I put my
modules in ``/etc/salt/_modules`` only, using the other path for States only.

Remember this structure, as this is what I'm going to refer to within this post.

Your first custom module
========================

Let's start with something easy. Say we define a function that always returns
boolean ``True``. In a file, say ``/etc/salt/_modules/example.py`` define a
simple Python function:

``/etc/salt/_modules/example.py``
```python
def first():
    return True
```

As you can notice, the code is linear, and similar to writing an arbitrary
Python scripts without any specifica.

After loading the module by executing 
[``saltutil.sync_modules``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.saltutil.html#salt.modules.saltutil.sync_modules),
you can go ahead an invoke the newly defined function:

```bash
$ salt 'your-device' example.first
your-device:
    True
```

Cross-calling Executon Modules
==============================

This section is already well detailed in 
[this](https://docs.saltstack.com/en/latest/ref/modules/#cross-calling-execution-modules)
section of the documentation, but I wanted to make sure you're not going to skip
it. I also wanted to reassure you that you can equally cross-call native
functions, as well as custom functions you define in your environment.

As an example, let's define two functions, say ``example.second`` and
``example.third`` that call the native 
[``test.false``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.test.html#salt.modules.test.false_)
Execution Function, and the ``example.first`` function defined above:

``/etc/salt/_modules/example.py``
```python
def second():
    return __salt__['test.false']()


def thrid():
    return __salt__['example.first']()
```

Again, after reloading the modules by executing ``saltutil.sync_modules``, you
can execute from the CLI:

```bash
$ salt 'your-device' example.second
your-device:
    False
$ salt 'your-device' example.thrid
your-device:
    True
```

*Remember*: always use the open-close braces (``{`` and ``)``) to properly
invoke the function -- ``__salt__`` is a mapping to all the available functions
Salt is aware of, each element pointing to the memory address where the function
is available for execution. Therefore, it is important to have the braces even
thugh you may not pass any arguments to the function, as in the previous
examples.

Salt functions for low-level API calls
--------------------------------------

The modules newly added in the 2019.2 Salt release, provide direct API call for
the following platforms:

- Arista, over eAPI, via the pyEAPI Python library.
- Cisco Nexus, over the NX-API (direct HTTP request)
- Junos, over NETCONF, through the existing 
  [junos](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.junos.html#module-salt.modules.junos)
  module.
- A variety of platforms, using screen-scraping over either SSH or Telnet (i.e.,
  issue CLI commands and return the text output, as you'd normally have on your
  terminal when executing the command(s) manually).
  You can find the complete list of supported platforms on [GitHub](https://github.com/ktbyers/netmiko#supports).

  While this is not encouraged usage, it is sometimes the only available choice,
  particularly on plaforms that fail to offer a (proper) or consistent API, but
  not limited to: for example, on Junos (and others) that have high coverage
  over the API for the available commands, there are cases when some are not
  currently supported. As an example, on Junos you cannot gather the MTR results
  to a destination over NETCONF; instead, using Netmiko you can collect these
  very easily by simply issuing the CLI command you'd normally use.

  This module might go very well together with the 
  [TextFSM](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.textfsm_mod.html)
  Execution Module added in the 2018.3.0 release, that allows raw text parsing
  though Google's [TextFSM](https://github.com/google/textfsm) Python library,
  and exposing Salt's simplicity for managing the templates in the local or
  remote environment. See the 
  [documentation](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.textfsm_mod.html)
  for more details and usage examples.

For NAPALM users, the following functions have been added to make API calls
without any further effort: once the modules are available, you can start using:

- Junos:
  - [``napalm.junos_rpc``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.junos_rpc): execute RPC calls and return the output as Python
    object.
  - [``napalm.junos_cli``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.junos_cli): execute CLI commands and return the output as either
    text or Python object.
  - [``napalm.junos_commit``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.junos_commit): for more flexible options when committing the
    configuration on Junos, that NAPALM doesn't currently have, e.g., commit
    confirmed, synchronise, force sync, etc.
- Arista:
  - [``napalm.pyeapi_run_commands``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.pyeapi_run_commands): execute a list of CLI / show style commands
    and return the output as text or Python object.
  - [``napalm.pyeapi_config``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.pyeapi_config): 
    configure the Arista switch with the specified commands.
- Cisco Nexus:
  - [``napalm.nxos_api_show``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.nxos_api_show): 
    execute one or more show commands and return the output as plain text or
    Python object.
  - [``napalm.nxos_api_config``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.nxos_api_config):
    configure the Nexus switch with the specified commands, via the NX-API.
- Any platform suported by Netmiko (including the ones above):
  - [``napalm.netmiko_commands``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.netmiko_commands): 
    send one or more CLI commands to be executed on the device, and return the
    output as plain text.
  - [``napalm.netmiko_config``](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.netmiko_config):
    Load one or more configuration commands on the network device. The commands
    can equally be loaded from a local or remote path, and passed through Salt's
    template rendering pipeline (by default using Jinja as the template
    rendering engine).

Let's take a look at some examples, from the command line:

### ``napalm.junos_cli``

Let's gather the RPKI statistics using the Juniper CLI command ``show validation statistics``
as we would execute manually from the CLI of the router:

```bash
$ salt 'juniper-router' napalm.junos_cli 'show validation statistics'
juniper-router:
    ----------
    message:
        
        Total RV records: 78242
        Total Replication RV records: 78242
          Prefix entries: 73193
          Origin-AS entries: 78242
        Memory utilization: 15058371 bytes
        Policy origin-validation requests: 0
          Valid: 0
          Invalid: 0
          Unknown: 0
        BGP import policy reevaluation notifications: 1925885
          inet.0, 1763165
          inet6.0, 162720
    out:
        True
```

The putput returned is plain text, as you'd normally get when executing manually
direactly on the device.

The ``napalm.junos_cli`` command is smart enough to execute RPC requests
even though you're invoking via the CLI command, by passing the ``format=xml``
argument, e.g,

```bash
$ salt 'juniper-router' napalm.junos_cli 'show validation statistics' format=xml
juniper-router:
    ----------
    message:
        ----------
        rv-statistics-information:
            ----------
            rv-statistics:
                ----------
                rv-bgp-import-policy-reevaluations:
                    1925885
                rv-bgp-import-policy-rib-name:
                    - inet.0
                    - inet6.0
                rv-bgp-import-policy-rib-reevaluations:
                    - 1763165
                    - 162720
                rv-memory-utilization:
                    15058761
                rv-origin-as-count:
                    78244
                rv-policy-origin-validation-requests:
                    0
                rv-policy-origin-validation-results-invalid:
                    0
                rv-policy-origin-validation-results-unknown:
                    0
                rv-policy-origin-validation-results-valid:
                    0
                rv-prefix-count:
                    73195
                rv-record-count:
                    78244
                rv-replication-record-count:
                    78244
    out:
        True
```

In this case, the output returned by ``napalm.junos_cli`` is a Python dictionary.
If you're not familiar with Salt yet, to convince yourself, let's take a look at
the raw object (the above is a visual display of the same, for a more human
readable output):

```bash
$ salt 'juniper-router' napalm.junos_cli 'show validation statistics' format=xml --out=raw
{'juniper-router': {'message': {'rv-statistics-information': {'rv-statistics': {'rv-bgp-import-policy-rib-reevaluations': ['1763165', '162720'], 'rv-policy-origin-validation-requests': '0', 'rv-policy-origin-validation-results-valid': '0', 'rv-origin-as-count': '78244', 'rv-memory-utilization': '15058761', 'rv-prefix-count': '73195', 'rv-record-count': '78244', 'rv-policy-origin-validation-results-unknown': '0', 'rv-bgp-import-policy-rib-name': ['inet.0', 'inet6.0'], 'rv-replication-record-count': '78244', 'rv-bgp-import-policy-reevaluations': '1925885', 'rv-policy-origin-validation-results-invalid': '0'}}}, 'out': True}}
```

### ``napalm.junos_rpc``

While the CLI command is easier to use (and morea readable / self-explanatory),
it is often better to invoke the RPC as that is the same cross-platform / cross-version,
while the CLI command is not guaranteed to be consistent. To find out the RPC
request to use, from the CLI of your router you can append ``| display xml rpc``
to the usual command, e.g.,

```
mircea@juniper-router> show validation statistics | display xml rpc 
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/18.1R3/junos">
    <rpc>
        <get-validation-statistics-information>
        </get-validation-statistics-information>
    </rpc>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>
```

The RPC in this case is ``get-validation-statistics-information``, and this is
what we're going to pass to the ``napalm.junos_rpc`` function:

```bash
$ salt 'juniper-router' napalm.junos_rpc get-validation-statistics-information
juniper-router:
    ----------
    comment:
    out:
        ----------
        rv-statistics-information:
            ----------
            rv-statistics:
                ----------
                rv-bgp-import-policy-reevaluations:
                    1925885
                rv-bgp-import-policy-rib-name:
                    - inet.0
                    - inet6.0
                rv-bgp-import-policy-rib-reevaluations:
                    - 1763165
                    - 162720
                rv-memory-utilization:
                    15058761
                rv-origin-as-count:
                    78244
                rv-policy-origin-validation-requests:
                    0
                rv-policy-origin-validation-results-invalid:
                    0
                rv-policy-origin-validation-results-unknown:
                    0
                rv-policy-origin-validation-results-valid:
                    0
                rv-prefix-count:
                    73195
                rv-record-count:
                    78244
                rv-replication-record-count:
                    78244
    result:
        True
```

### ``napalm.pyeapi_run_commands``

In a similar way, we can invoke show-like commands on Arista switches, and
the ``napalm.pyeapi_run_commands`` accepts one or more commands to be executed,
by default returning the output as Python object, e.g.,

```bash
$ salt 'arista-switch' napalm.pyeapi_run_commands 'show version'
arista-switch:
    |_
      ----------
      architecture:
          i386
      bootupTimestamp:
          1534844216.0
      hardwareRevision:
          11.03
      internalBuildId:
          5c08e74b-ab2b-49fa-bde3-ef7238e2e1ca
      internalVersion:
          4.20.8M-9384033.4208M
      isIntlVersion:
          False
      uptime:
          18767827.61
      version:
          4.20.8M
```

To execute more than one operational command, pass them sequentially, e.g.,
``salt 'arista-switch' napalm.pyeapi_run_commands 'show lldp neighbors' 'bash timeout 10 ping 1.1.1.1 -c 5'``.

To receive the output as text (instead of Python object), set the argument
``encoding`` as ``text``, e.g.,
``salt 'arista-switch napalm.pyeapi_run_commands 'show version' encoding=text``.

### ``napalm.netmiko_commands``

I'm going to skip the other functions mentioned earlier and focus on the Netmiko
functions for a little. Firstly, let's take a look at some examples when using
Netmiko through the ``napalm`` Salt module, even when working with platforms
already covered and provide good API support (though not 100% complete), such as
Arista, e.g.,

```bash
$ salt 'arista-switch' napalm.netmiko_commands 'show version'
arista-switch:
    - Hardware version:    11.03
      
      Software image version: 4.20.8M
      Architecture:           i386
      Internal build version: 4.20.8M-9384033.4208M
      Internal build ID:      5c08e74b-ab2b-49fa-bde3-ef7238e2e1ca
      
      Uptime:                 31 weeks, 0 days, 5 hours and 28 minutes
```

Which this usage may be superfluous in this particular case, this function can
be very handy when you require the output of a specific command, that is not
available over the API. For example, on Junos platforms I needed to collect MTR
results automatically, but the ``traceroute monitor`` command is not available
over NETCONF. So here's how ``napalm.netmiko_commands`` can save the day:

```bash
$ salt 'juniper-router' napalm.netmiko_commands 'traceroute monitor 1.1.1.1 summary'
juniper-router:
    - 
      HOST: juniper-router              Loss%   Snt   Last   Avg  Best  Wrst StDev
        1. one.one.one.one               0.0%    10    0.6   1.0   0.5   4.2   1.1
```

While on platforms that provide good API support such as Junos or Arista
``napalm.netmiko_commands`` is only used in corner cases, it's one of the few
options you can have when working with operating systems such as Cisco IOS,
IOS-XR, and many others.

In a similar way to ``napalm.pyeapi_run_commands``, you can pass multiple CLI
comamnds to execute and collect their text output.

As a side note, in order to process that output further you might prefer using
the [TextFSM](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.textfsm_mod.html)
module natively integrated into Salt starting with release 2018.3.0, which is
also very easy to use.

### ``napalm.netmiko_config``

This function can help you send configuration commands to the targeted device,
through Netmiko (thus over SSH). You can equally load one or more configuration
commands, and it reproduces the behaviour of an user that would normally be
configuring the device manually, e.g.,

```bash
$ sudo salt 'arista-swtich' napalm.netmiko_config 'ntp server 10.10.10.1'
arista-swtich:
    config term
    arista-swtich(config)#ntp server 10.10.10.1
    arista-swtich(config)#end
    arista-swtich#
```

One particular note to kepe in mind with this command is that on platforms such
as Juniper, where you need to commit the configuration, and transfer the changes
into the running config, you will need to explicitly pass the argument
``commit=True``, otherwise it's going to discard the changes. It is always a
good practice to use the ``commit`` argument whenever you are absolutely
confident that the changes are correct:

```bash
$ salt 'juniper-router' napalm.netmiko_config 'set system ntp server 10.10.10.1'
juniper-router:
    configure 
    Entering configuration mode
    
    [edit]
    salt@juniper-router# set system ntp server 10.10.10.1 
    
    [edit]
    salt@juniper-router# exit configuration-mode 
    The configuration has been changed but not committed
    Exiting configuration mode
    
    salt@juniper-router> 
```

The correct usgae for the above would be:

```bash
$ salt 'juniper-router' napalm.netmiko_config 'set system ntp server 10.10.10.1' commit=True
juniper-router:
    configure 
    Entering configuration mode
    The configuration has been changed but not committed
    
    [edit]
    salt@juniper-router# set system ntp server 10.10.10.1 
    
    [edit]
    salt@juniper-router# commit 
    commit complete
    
    [edit]
    salt@juniper-router#
```

That said, I am not aware of any use case where you'd require Netmiko for
configuration management on Juniper platforms, but I wanted to share this
however, as Netmiko covers a large variety of platforms, and you might find
this helpful at some point.

Calling low-level API functions from your custom functions
==========================================================

To be able to fully understand this section, I would need you to make sure the
previous paragraphs are entirely clear to you.

Looking again at the _Cross-calling Executon Modules_ paragraph, you can invoke
any of the functions previously presented, from your custom module. For example,
let's start with something very simple and write a function that returns the
operating system version. For an Arista switch, the code would be as simple as:

``/etc/salt/_modules/example.py``
```python
def version():
    res = __salt__['napalm.pyeapi_run_commands'](‘show version’)
    return res[0]['version']
```

As ``napalm.pyeapi_run_commands`` can execute one or more commands, the output
is a list, which is why you need to do a lookup at index ``0``, then return the
operating system version under the ``version`` key. To check the exact structure
and know exactly what keys to lookup for, I always execute the equivalent CLI
command checking the raw Python output:

```bash
$ salt 'arista-switch' napalm.pyeapi_run_commands 'show version' --out=raw
{'arista-switch': [{'uptime': 6776189.66, 'internalVersion': '4.20.8M-9384033.4208M', 'architecture': 'i386', 'bootupTimestamp': 1534844216.0, 'version': '4.20.8M', 'isIntlVersion': False, 'internalBuildId': '5c08e74b-ab2b-49fa-bde3-ef7238e2e1ca', 'hardwareRevision': '11.03'}]}
```

*Note*: The example above has been crafted for examplifcation purpose only, to
be easier to be understood, and focus on how to achieve something very easy. In
this particular case, if you need to check the running version, you can this
information into th Grains, and access it from a module as
``__grains__['version']``. The example stands however when you would need to
gather other pieces of information from the network device, that are not present
into the Grains.

Once saved and synchronsed, you can execute as:

```bash
$ salt 'arista-switch' example.version
arista-switch:
    4.20.8M
```

Cross-platform implementation
-----------------------------

The features presented work independently for every platform, and their output
is similarly different (as the network device presents it, over the API of
choice). Now, in order to preserve one of the most important adavantages of
NAPALM, you have to abstract away the functionality, so the end users are
able to use the exact syntax regadless of the platform they are targeting. There
are two good ways do do so, and it mainly depends on personal preference which
one you prefer.

Say we continue the previous example, and write a function that returns the
operating system version, while preserving the same syntax when invoking the
function. In both cases, the key is using the ``os`` Grain which tells you what
operating system are you executing against.

### Abstract away inside the function

I'll firstly provide the code example, then explain below:

``/etc/salt/_modules/example.py``
```python
def version():
    if __grains__['os'] == 'junos':
        ret = __salt__['napalm.junos_cli']('show version', format='xml')
        return ret['message']['software-information']['junos-version']
    elif __grains__['os'] == 'eos':
        ret = __salt__['napalm.pyeapi_run_commands']('show version')
        return ret[0]['version']
    elif __grains__['os'] == 'nxos':
        ret = __salt__['napalm.nxos_api_rpc']('show version')
        return ret[0]['result']['body']['sys_ver_str']
    return 'Not supported on this platform'
```

What the code does, is simply checking the operating system name, by inspecting
the ``os`` Grain, and then invoke the appropriate Salt function. This way, when
the function is executed against a Juniper device, it's going to cross-call
``napalm.junos_cli`` then do the lookup under the ``message``,
``software-information``, and ``junos-version`` keys (do check the raw output to
make sure you understand why), while against an Arista switch, it's invoking
``napalm.pyeapi_run_commands`` as explained previously.

### Separate modules per platform

While in the above the logic per platform is implemented inside the same
function, when we have to put a lot more business logic, the code may get too
complex or harder to follow (arguably, again, depends on personal preference).

The alternative is splitting the code into separate files, one per platform.
This is using Salt's feature for registering different physical modules under
the same virtual name. Check out the documentation for [virtual 
modules](https://docs.saltstack.com/en/latest/ref/modules/#virtual-modules).

The core idea is that you create one file for each platform, and declare the
same virtual name. Say we want to continue using the module name from the
previous examples, ``example``, then the files would be called
``example_junos.py``, ``example_eos.py``, and ``example_nxos.py``. Note, you
can choose whatever filename you may prefer, the platform availability is made
inside the ``__virtual__`` function:

``/etc/salt/_modules/example_junos.py``
```python
def __virtual__():
    if __grains__['os'] == 'junos':
        return 'example'
    else:
        return (False, 'Not loading this module, as this is not a Junos device')


def version():
    ret = __salt__['napalm.junos_cli']('show version', format='xml')
    return ret['message']['software-information']['junos-version']
```

The ``__virtual__`` function returns ``example`` (which is the virtual name of
the module, i.e., the name you're going to use on the CLI, or other Execution
Modules), but only when the ``os`` Grain is ``junos``, otherwise, the code from
``example_junos.py`` will not be loaded (and it may load the code from a
different physical module instead).
Let's take a look at the equivalent module for ``eos``:

``/etc/salt/_modules/example_eos.py``
```python
def __virtual__():
    if __grains__['os'] == 'eos':
        return 'example'
    else:
        return (False, 'Not loading this module, as this is not an Arista switch')


def version():
    ret = __salt__['napalm.pyeapi_run_commands']('show version')
    return ret[0]['version']
```

Both options are good, you can use whichever you feel more comfortable to with.
I use both, depending on the complexity of the code.

Happy hacking
=============

Using this methodology, I've been able to significantly speed up the development
process, and make it more enjoyable at the same time. It goes without saying
that you would still be able to help the community with your contributions by
submiting the modules to the
[official Salt codebse](https://github.com/saltstack/salt) so they will be
available in the next Salt release, or into the [salt-contrib repository](https://github.com/saltstack/salt-contrib),
[napalm-salt](https://github.com/napalm-automation/napalm-salt), or even your
own repository (and please let everyone know -- if you prefer, let me know, and
I would be happy spread the word for awareness).

Beyond the modules I have expanded on, you also have an army of few other
features such as the 
[ciscoconfparse module](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.ciscoconfparse_mod.html)
for the [ciscoconfparse Python library](http://www.pennington.net/py/ciscoconfparse/index.html),
[iosconfig](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.iosconfig.html#module-salt.modules.iosconfig) 
for transforming Cisco IOS-like text-based configuration style into Python
structures to be easier to work with. These are similarly integrated within the
``napalm`` Execution Module for ease of use -- see for example
[napalm.config_filter_lines](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.config_filter_lines),
[napalm.config_tree](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.config_tree), or
[napalm.scp_put](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.scp_put) 
and 
[napalm.scp_get](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_mod.html#salt.modules.napalm_mod.scp_get) 
which can help you implement the business logic you need. In
the end, I'd also like to invite you to take a look at the 
[PeeringDB module](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.peeringdb.html#module-salt.modules.peeringdb),
as well as the [module for 
NetBox](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.netbox.html#module-salt.modules.netbox),
which now has many more features available. I hope it's pretty clear that now
it is easier than ever to get started to automate and write code for your
network, by simply reusing and invoking features you have at your fingertips.

If you have a story to share in this direction, I'd love to hear it, and learn
from your experience. 
