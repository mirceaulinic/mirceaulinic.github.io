---
layout: post
title: Installing SaltStack and NAPALM
---

During the live session of the [NANOG tutorial](https://mirceaulinic.net/2017-02-14-network-automation-tutorial/), I have expanded on the installation side. However, among the feedback received from the readers that have been following the slides, people suggested that I should have included more details. This is why today I am going to provide more information for these topics: how to install the tools a network engineer requires to start automating.

Install NAPALM
--------------

NAPALM is available on [PyPI](https://pypi.python.org/pypi) (Python Package Index) which is the official repository for third-party Python libraries.

Historically the library has been a single package containing a separate class for each driver.
In 2016 we have decided that the best for the library's future is to split it into indenepdent sub-libraries, one for each driver. The driver name depends on the platform supported, e.g: `napalm-junos` is the driver implemeting the features for JunOS, `napalm-iosxr` for Cisco IOS-XR etc.

Without diving into further details, for the moment, let's note that each driver requires [napalm-base](https://github.com/napalm-automation/napalm-base).

The greatest advantage of splitting the library into multiple ancestors is that the user is able to install only the package(s) they require for their network. They can be installed using `pip`, which is the most used Python package manager, for example: `pip install napalm-iosxr napalm-eos napalm-ios`.

You are still able to install everything, by doing: `pip install napalm`. But given that the number of drivers supported is increasing, so does the number of sub-libraries; therefore this is not quite a good practice!
Another point I would like to make is using the [Recursive Upgrade option](https://pip.pypa.io/en/stable/user_guide/#only-if-needed-recursive-upgrade). In the NAPALM community we have agreed to release very often minor releases, providing bug fixes, so doing `pip install -u <package_name>` would probably help you in many circumstances.

Unfortunately NAPALM is not available as system packages. This is a major downside, as many of the underlying libraries have different platform dependencies. For example, to be able to use `napalm-junos` on a Debian, as [junos-eznc](https://github.com/Juniper/py-junos-eznc) is the main underlying library, you will need to install several packages: python-cffi, python-dev, libxslt1-dev, libssl-dev, libffi-dev etc. For more complete details, refer to [NAPALM installation dependencies](http://napalm.readthedocs.io/en/latest/installation.html#dependencies). The existing document does not cover all possible operating systems, so pull requests are greately appreciated!

Install SaltStack
-----------------

In order to install the latest SaltStack release, the reference document can be found at: [https://docs.saltstack.com/en/latest/topics/installation/](https://docs.saltstack.com/en/latest/topics/installation/)

Below I will present the require steps to install on [Debian Jessie](https://docs.saltstack.com/en/latest/topics/installation/debian.html):

- Add Jessie backports to apt sources: `sudo echo 'deb http://httpredir.debian.org/debian jessie-backports main' >> /etc/apt/sources.list`
- Add SaltStack repo to apt sources: `sudo echo 'deb http://repo.saltstack.com/apt/debian/8/amd64/latest jessie main' >> /etc/apt/sources.list.d/saltstack.list`
- Download and add the SaltStack repo key: `wget -O - https://repo.saltstack.com/apt/debian/8/amd64/latest/SALTSTACK-GPG-KEY.pub | sudo apt-key add - `
- Update the sources: `sudo apt-get update`
- install the salt-master package: `sudo apt-get install salt-master`
- install salt-minion which includes also the salt-proxy: `sudo apt-get instal salt-minion`

### Note

In the example above I have used the official SaltStack repository. The Debian repo is equally good, but usually the packages for new releases are avaialble with a delay of several delays.
