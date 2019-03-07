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
subclassing NAPALM's native classes (i.e., JunOSDriver, EOSDriver, etc.) and
extending them by adding more methods I needed. While this approach was easy
enough, after writing the code, there were two additional steps required:
1. release a new version of this custom library, and 2. install it. While both
are easy enough in theory, when releasing code often (several times a day)
it can get tedious, especially when you have to mess with Python's poor
packaging system, and, on top of that, it is little point to do so when using
Salt which provides mature ways of spreading out new code out of the box. Salt's
native filesystem can be used to release new code in the form of Execution
Modules. In the latest Salt release, 2019.2.0 - codename Fluorine, there are
many new features that expose the low level API that NAPALM is using (and beyond
that), which you can invoke from your own custom modules to implement the
business logic you need. The implementation is much easier that it sounds.
Check out the [release notes](https://docs.saltstack.com/en/develop/topics/releases/2019.2.0.html#napalm)
to before going any further.

*Note*: If you are not running 2019.2.0 yet, dont' worry, you are covered: all
you have to do in order to follow when I'm descirbing in the steps below, is to
copy the files directly from GitHub. For example, one of the files required is
[napalm_mod.py](https://github.com/saltstack/salt/blob/2019.2/salt/modules/napalm_mod.py).
To be able to use this module with older Salt releases, simply copy this file
into one of your ``file_roots`` directory. For example, if one of the
``file_roots`` directory is ``/etc/salt/extmods``, then you only have to
execute, e.g.,

```bash
$ cd /etc/salt/extmods
$ mkdir _modules && cd _modules/
$ wget https://raw.githubusercontent.com/saltstack/salt/2019.2/salt/modules/napalm_mod.py
```

*Remember* to always use ``wget`` with the _raw_ link from GitHub, otherwise
you'll download an HTML file instead. :-)

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
---------------------------------

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


