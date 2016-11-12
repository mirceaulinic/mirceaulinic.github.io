---
layout: post
title: Network orchestration with Salt and NAPALM
subtitle: Because rendering templates is simply not enough
---

## Introduction

Automation is the new buzzword in networking today. By automation many people understand a configuration management system, i.e. keeping the configuration of their devices consistent across their networks. Which is perfectly fine if that's all you want from your devices, it is already a big step forward. But there are many use cases when you need much more than that! Therefore, in order to avoid any ambiguity, we should use a different term: orchestration - which is much more... it is what SREs have been doing for years on servers. Of course configuration management is one of the required features, but not limited to: retrieving essential information about your devices and reacting are equally important.

## Motivation

To achieve these goals, for sure you need presence of mind and firstly draw your requirements. Doing what your neighbour/friend does, is not enough reason to repeat the same mistakes and discover after several months that your tools cannot do the job you actually need.

For network orchestration, I recommend two ingredients: Salt and NAPALM.

### NAPALM

NAPALM stands for _Network Automation and Programmability Abstraction Layer with Multivendor support_ is a Python library build by a community of passionate network engineers. It is a wrapper of different libraries that communicate directly with the network device. It has grown very fast and currently provides support for quite a few operating systems, most used being: JunOS, IOS-XR, EOS, IOS, NX-OS etc. - complete list at: [http://napalm.readthedocs.io/en/latest/support/index.html](http://napalm.readthedocs.io/en/latest/support/index.html).

### Salt

Without diving into extensive details and comparisons, Salt is the most complete framework you can get for free. On the other hand is good to keep in mind that the perfect software was not invented yet (and never will, in my opinion) - in order to make the right choice, you definitely need to know your goals and determine what product can accomplish them (or at least most of them). Right now, the major drawback of Salt is the setup (but will be simplified very soon). I provided some easier instruction notes in [the dedicated repo under NAPALM](https://github.com/napalm-automation/napalm-salt) having as target users the network engineers interested in this software; but after you have it up and running, everything is natural and the documentation is just great. This is also because the syntax cannot have any ambiguities, e.g. it is quite obvious that executing ```snmp.config``` would provide the configuration of SNMP, ```ntp.peers``` the list of NTP peers configured on the device etc.

At Cloudflare we had certain requirements solved thanks to the following features:

- Find reliably and fast relevant details (e.g. MAC addresses, IP addresses, interfaces given a circuit ID, VLAN IDs etc., looking into the ARP tables, MAC address tables, LLDP neighbors and other sources of information, from all network devices)
- Schedule jobs to make sure the config is consistent (without running manually a command; yes, the system will do that for you!)
- Parallel execution (because that's what computers are good at, aren't they?)
- Abstracted & vendor-agnostic configurations
- Native cache, without requiring us to deploy and maintain complex database systems (e.g. LLDP neighbors details, which don't change very often, you can cache the data and you can access it instantly whenever needed)
- REST API
- High availability
- GPG encryption
- Pull from Git or SVN

We have mixed them together and NAPALM will be fully integrated in the core of Salt beginning with release ```2016.11```: available documentation can be explored starting from [the proxy module](https://docs.saltstack.com/en/develop/ref/proxy/all/salt.proxy.napalm.html). This is very good news as the setup process will be highly simplified - you only need to [install](https://docs.saltstack.com/en/latest/topics/installation/) (usually one command, e.g. for Debian: ```sudo apt-get install salt-master```) & use.

#### Architecture

The general picture is hub and spoke with a master controlling several _minions_. On the server side, on each minion server is installed a software package called _salt-minion_. This is currently impossible on all network devices, vendors still prefer to keep their platforms closed, not allowing to install custom software directly.

For these reasons, Salt introduced the proxy minion. The proxy is not a different platform, it is just another process forked per each minion representing the network device, emulating the behavior of a salt-minion. Basically it is still a hub and spoke pattern, but with virtual minions.

It also worths looking at the speed: the connection is established just once and kept open; anytime you need to execute a command, the session is ready to deliver the data requested almost instantly.

## How to use?

Assuming the environment is setup and ready to be used (for example [these notes](https://github.com/napalm-automation/napalm-salt)), you are now ready to define the first device.

Under the directory specified as ```file_roots``` (default is ```/etc/salt/states```) in the [master config file](https://github.com/napalm-automation/napalm-salt/blob/master/master) create the SLS  descriptor as specified in the [napalm proxy documentation](https://docs.saltstack.com/en/develop/ref/proxy/all/salt.proxy.napalm.html), say we call it ```edge01_bjm01.sls``` corresponding to hostname ```edge01.bjm01```.

In the [top.sls](https://docs.saltstack.com/en/latest/ref/states/top.html) file under the same directory must include the file defined above and map the name of the file with the minion ID you want:

```yaml
base:
  edge01.bjm01:
    - edge01_bjm01
```

Which tells Salt that the minion ```edge01.bjm01``` has associated the file descriptor ```edge01_bjm01``` (without the ```.sls``` extension!). The minion ID does not need to correspond with the hostname or the filename, but it's a good practice in order to avoid mistakes!

**After each update of the top file, the salt-master process needs to be restarted**. To do so, the process can be easier controlled using [systemctl](https://github.com/napalm-automation/napalm-salt#running-the-master-as-a-service):

```bash
$ sudo systemctl restart salt-master
```

[Similarly](https://github.com/napalm-automation/napalm-salt#running-the-proxy-minion-as-a-service) for the proxy minion process, using the minion ID, as specified in the top file:

```bash
$ sudo systemctl start salt-proxy@edge01.bjm01
```

Logs can be found usually under ```/var/log/salt/master``` and ```/var/log/salt/proxy```. The level can be set in the master config file specifying the desired level as ```log_level_logfile``` (default is ```warning```). To store the logs in a different location, set the custom value for ```log_file``` - [see more](https://github.com/napalm-automation/napalm-salt/blob/master/master#L392-L431).

After the proxy minion process is started, you are now able to execute:

```bash
$ sudo salt edge01.bjm01 net.connected
```

If everything is fine and the connection succeeded, will return ```True```. Observe the command syntax begins with the keyword ```salt``` followed by the minion ID and then the name of the function.

Similarly, can be executed the commands from the [NET](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_network.html#module-salt.modules.napalm_network) execution module, for example ```net.arp``` which returns the ARP table:

```bash
$ sudo salt edge01.bjm01 net.arp
```

You can also display in the output you need (e.g. YAML):

```bash
$ sudo salt --out=yaml edge01.bjm01 net.arp
```

The complete list of available output formats can be found [here](https://docs.saltstack.com/en/latest/ref/renderers/#full-list-of-renderers).

Exactly in the same manner can be used for the other available modules (examples in the documentation): [BGP](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_bgp.html#module-salt.modules.napalm_bgp), [NTP](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_ntp.html#module-salt.modules.napalm_ntp), [SNMP](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_snmp.html#module-salt.modules.napalm_snmp), [Users](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_users.html#module-salt.modules.napalm_users), [Route](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_route.html#module-salt.modules.napalm_route) etc.

In the examples above we have been working with one single minion for simplicity. Moving forward, let's introduce the [grains](https://docs.saltstack.com/en/develop/ref/grains/all/salt.grains.napalm.html#module-salt.grains.napalm) -- they help you group devices based on their characteristics. The collection of this information is immediately after the connection is established. Few examples:

- Execute traceroute on all minions whose ID begins with ```edge``` (yes, you can use regex to match):

```bash
$ sudo salt 'edge*' net.traceroute 8.8.8.8
```

- Execute the raw CLI command ```show version``` on all devices running JunOS:

```bash
$ sudo salt -G 'os:junos' net.cli "show version"
```

- Retrieve the ARP table from all Cisco routers running IOS-XR 6.0.2:

```bash
$ sudo salt -G 'os:iosxr and version:6.0.2' net.arp
```

- Retrieve the RPM probes results from all Juniper MX480 routers:

```bash
$ sudo salt -G 'model:MX480' probes.results
```

- Display the serial number from all devices whose serial number begins with ```FOX```:

```bash
$ sudo salt -G 'serial:FOX*' grains.get serial
```

## Conclusion

In this blog post I have covered the very basic setup details and a brief introduction to the command syntax and available execution modules. Salt is a swiss knife, very complex, with tons of features ready to help: there are about [20 more](https://docs.saltstack.com/en/latest/ref/index.html) types of modules to be covered. The most important will be explained and exemplified soon!
