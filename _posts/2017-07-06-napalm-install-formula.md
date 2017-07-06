---
layout: post
title: Installing NAPALM using Salt
subtitle: The easy way
---

In one of my previous posts, I have presented [how to install NAPALM and Salt](https://mirceaulinic.net/2017-03-14-install-saltstack/). While Salt can be installed directly from the [SaltStack Repo](https://repo.saltstack.com/), very easy, using [Salt Bootstrap](https://docs.saltstack.com/en/latest/topics/tutorials/salt_bootstrap.html), NAPALM required more steps and there are a few [system dependencies](http://napalm.readthedocs.io/en/latest/installation/index.html#dependencies) you may need to consider. For Salt users there's an easier way to get everything installed using one simple command.

Salt Forumlas
-------------

Formulas are pre-written Salt States, provided by the Salt community. There are plenty of formulas available unde the [SaltStack Formulas GitHub organization](https://github.com/saltstack-formulas). Their installation is easy, well explained in detail here: [https://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html#installation](https://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html#installation). Basically all you need to do is write your pillar as specified in the formula; each formula has also an ``pillar.example`` file to help with a structure example for the pillar.

For NAPALM users, there's a new Salt forumla: [napalm-install-formula](https://github.com/saltstack-formulas/napalm-install-formula). Using this formula, we can install the system packages very easily, in addition to the ``PyPi`` requirements for the NAPALM drivers. This is also a great way to keep your environment updated, by executing a command as simple as ``salt-call state.sls napalm_install`` (I will explain below).

Regular minions and proxy minions
---------------------------------

The major difference between regular minions and proxy minions is that the latter are just one process (per network device managed), able to run from everywhere, as long as they are able to contact the Salt master. To use the ``napalm-install-formula``, you need to run the ``salt-minion`` on the same machine where you have the proxy minion(s), connected eventually to the same Salt master. This does not bring an extra dependency, the ``salt-proxy`` binary is anyway included in the ``salt-minion`` package; in other words, to be able to run proxy minions, you have already installed the ``salt-minion``, but it might not be used.

We will need to start the Salt minion, as this is the process managing our server, while the proxy minion manages our network gear.

Configure and start the ``salt-minion``
---------------------------------------

The regular minion is very simple to be started, less complicated than the proxy:

1. Specify the host of the Salt master in the minion configuration file (typically under ``/etc/salt/minion`` or ``/srv/minion``), under the configuration field ``master``. In this example, my minion is using the ``salt-master`` running on the same host (in my lab I run the proxy minions on the same physical machine as the master):

```yaml
master: localhost
```

2. Start the minion process:

```bash
$ sudo systemctl start salt-minion
```

3. Check the unaccepted keys:

```bash
$ sudo salt-key -L
Accepted Keys:
csr1
device1
device2
nxos-spine1
vmx1
Denied Keys:
Unaccepted Keys:
eos
ip-172-31-11-15
Rejected Keys:
```

We can notice that the minion ``ip-172-31-11-15`` is not accepted yet by the master. Accept the key: ``$ sudo salt-key -a ip-172-31-11-15``.

In the following examples, ``ip-172-31-11-15`` is the minion ID of the Salt minion that manages the server where the proxies run.

Configure the ``napalm-install-formula``
----------------------------------------

1. The pillar has the following structure:

``/etc/salt/pillar/napalm.sls``
```yaml
napalm:
  install:
    - napalm-iosxr
    - napalm-junos
```

Where we can list the NAPALM drivers we need. To install the complete library, with all the possible drivers, you can simply list ``napalm``.

2. Map the ``napalm.sls`` pillar defined above to the Salt regular minion, idendified by its ID:

``/etc/salt/pillar/top.sls``
```yaml
base:
  'ip-172-31-11-15':
    - napalm
```

3. Refresh pillar on the local minion:

```bash
$ sudo salt-call saltutil.refresh_pillar
local:
    True
```

When executing from the master, the equivalent command is: ``$ sudo salt 'ip-172-31-11-15' saltutil.refresh_pillar``.

4. Check that the minion has the right data:

```yaml
$ sudo salt-call pillar.get napalm:install
local:
    - napalm-junos
    - napalm-iosxr
```

5. If you have installed the ``napalm-install-formula`` correctly, you should be able to execute: ```$ sudo salt-call state.show_sls napalm_install```. Otherwise, make sure you have the [map.jinja](https://github.com/saltstack-formulas/napalm-install-formula/blob/master/napalm_install/map.jinja) and [init.sls](https://github.com/saltstack-formulas/napalm-install-formula/blob/master/napalm_install/init.sls) in a directory called ``napalm_install``, under one of the paths listed as [``file_roots``](https://docs.saltstack.com/en/latest/ref/configuration/master.html#file-roots) in the master configuration.

6. Execute a dry run and check the output (this step is not mandatory, but it is a good practice):

```bash
$ sudo salt-call state.sls napalm_install test=True
local:
----------
          ID: install_napalm_junos_pkgs
    Function: pkg.installed
      Result: None
     Comment: The following packages would be installed/updated: libssl-dev, python-cffi, libxslt1-dev, libffi-dev, python-dev
     Started: 11:45:03.200592
    Duration: 332.72 ms
     Changes:
----------
          ID: napalm-junos
    Function: pip.installed
      Result: None
     Comment: Python package napalm-junos is set to be installed
     Started: 11:45:03.775327
    Duration: 974.406 ms
     Changes:
----------
          ID: install_napalm_iosxr_pkgs
    Function: pkg.installed
      Result: None
     Comment: The following packages would be installed/updated: libssl-dev, python-cffi, python-dev, libffi-dev
     Started: 11:45:04.749919
    Duration: 9.268 ms
     Changes:
----------
          ID: napalm-iosxr
    Function: pip.installed
      Result: None
     Comment: Python package napalm-iosxr is set to be installed
     Started: 11:45:04.759294
    Duration: 957.672 ms
     Changes:

Summary for local
------------
Succeeded: 4 (unchanged=4)
Failed:    0
------------
Total states run:     4
Total run time:   2.274 s
```

The test mode tells us that there are several system packages to be installed, including ``libxslt1-dev``, ``libssl-dev`` together with the NAPALM libraries from PyPi, using ``pip``. As everything looks good, we can go ahead and install:

```bash
$ sudo salt-call state.sls napalm_install
local:
----------
          ID: install_napalm_junos_pkgs
    Function: pkg.installed
      Result: True
     Comment: 5 targeted packages were installed/updated.
              The following packages were already installed: python-pip, libxml2-dev
     Started: 11:47:43.398503
    Duration: 6123.864 ms
     Changes:
              ----------
              libffi-dev:
                  ----------
                  new:
                      3.1-2+deb8u1
                  old:
              libffi6:
                  ----------
                  new:
                      3.1-2+deb8u1
                  old:
                      3.1-2+b2
              libssl-dev:
                  ----------
                  new:
                      1.0.1t-1+deb8u6
                  old:
              libxslt-dev:
                  ----------
                  new:
                      1
                  old:
              libxslt1-dev:
                  ----------
                  new:
                      1.1.28-2+deb8u3
                  old:
              python-cffi:
                  ----------
                  new:
                      0.8.6-1
                  old:
              python-dev:
                  ----------
                  new:
                      2.7.9-1
                  old:
              python-dev:any:
                  ----------
                  new:
                      1
                  old:
----------
          ID: napalm-junos
    Function: pip.installed
      Result: True
     Comment: All packages were successfully installed
     Started: 11:47:50.485667
    Duration: 2536.705 ms
     Changes:
              ----------
              napalm-junos==0.11.0:
                  Installed
----------
          ID: install_napalm_iosxr_pkgs
    Function: pkg.installed
      Result: True
     Comment: All specified packages are already installed
     Started: 11:47:53.023603
    Duration: 4.962 ms
     Changes:
----------
          ID: napalm-iosxr
    Function: pip.installed
      Result: True
     Comment: All packages were successfully installed
     Started: 11:47:53.028663
    Duration: 4820.892 ms
     Changes:
              ----------
              napalm-iosxr==0.5.1:
                  Installed

Summary for local
------------
Succeeded: 4 (changed=2)
Failed:    0
------------
Total states run:     4
Total run time:  13.486 s
```

The state execution correctly setup the environment for NAPALM and upgraded the system dependencies to the latest releases.

From here on, if we need to install another NAPALM driver, but we are unsure about its dependencies, we can simply list it in the pillar, e.g.:

```yaml
napalm:
  install:
    - napalm-junos
    - napalm-iosxr
    - napalm-panos
```

Then execute the state:

```bash
$ sudo salt-call state.sls napalm_install
local:
----------
          ID: install_napalm_junos_pkgs
    Function: pkg.installed
      Result: True
     Comment: All specified packages are already installed
     Started: 11:52:17.193931
    Duration: 323.716 ms
     Changes:
----------
          ID: napalm-junos
    Function: pip.installed
      Result: True
     Comment: Python package napalm-junos was already installed
              All packages were successfully installed
     Started: 11:52:17.758566
    Duration: 2528.329 ms
     Changes:
----------
          ID: install_napalm_iosxr_pkgs
    Function: pkg.installed
      Result: True
     Comment: All specified packages are already installed
     Started: 11:52:20.287092
    Duration: 5.199 ms
     Changes:
----------
          ID: napalm-iosxr
    Function: pip.installed
      Result: True
     Comment: Python package napalm-iosxr was already installed
              All packages were successfully installed
     Started: 11:52:20.292391
    Duration: 1452.512 ms
     Changes:
----------
          ID: install_napalm_panos_pkgs
    Function: pkg.installed
      Result: True
     Comment: All specified packages are already installed
     Started: 11:52:21.745097
    Duration: 5.286 ms
     Changes:
----------
          ID: napalm-panos
    Function: pip.installed
      Result: True
     Comment: All packages were successfully installed
     Started: 11:52:21.750481
    Duration: 4880.599 ms
     Changes:
              ----------
              napalm-panos==0.4.0:
                  Installed

Summary for local
------------
Succeeded: 6 (changed=1)
Failed:    0
------------
Total states run:     6
Total run time:   9.196 s
```

If you are running a large scale network and your proxy minions are distributed across multiple servers, the dependencies installation becomes very stright forward:

```bash
$ sudo salt -N proxy-minions-servers state.sls napalm_install
```

Where ``proxy-minions-servers`` is a [node group](https://docs.saltstack.com/en/latest/topics/targeting/nodegroups.html) defined on the master, selecting the servers running the proxy processes.

Conclusions
-----------

Although there are some additional steps to get the formula up and running, on the long run this method will turn to be very handy and a good way to maintain your environment updated. In the NAPALM community, we have agreed to release bugfixes and feature releases as often as possible, so most of the time upgrading to the latest version will probably help. So why not use the [Salt schedule](https://docs.saltstack.com/en/latest/topics/jobs/) to execute the ``napalm_install`` state periodically and ensure your environment is always running the latest versions.
