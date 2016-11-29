---
layout: post
title: Salt 2016.11.0 (Carbon) released
bigimg: /img/carbon.png
---

The wait is over, NAPALM is finally fully integrated in an automation framework. Beginning with 2016.11 (Carbon), all the NAPALM features are integrated in the core of Salt - more details in the [release notes](). That means, when you install Salt you have by default the network automation features included!

## Why is this so important?

This is the result of 11 months of assiduous work and contributions to both NAPALM and Salt projects. Basically after less than one year, the network engineers finally have something they can just use without geting lost in complex setups that require deeper software knowledge - only two simple steps required: 1. Install & 2. Use.

### How did it look like before?

11 months ago we decided to go on the path of network automation having the goals desribed in &para; [*Salt*](https://mirceaulinic.net/2016-11-17-network-orchestration-with-salt-and-napalm/) but very soon we learned that none of the tools available at that time was able to help us as we needed! Because of this, we decided to build a system for us and also contribute to two great projects: Salt and NAPALM, allowing others in the future to overcome these shortfalls.

#### NAPALM

NAPALM eveloved a lot over the last months: at the beginning of the year there were only [6 features available](https://github.com/napalm-automation/napalm/blob/f89d489f223a049db513aed69c3d000cec062b5c/docs/support/index.rst#getters-support-matrix) versus now, [at the end of November](https://github.com/napalm-automation/napalm/blob/4cf701a53ab166317d3b8a635392b9314c6b23d1/docs/support/index.rst#getters-support-matrix). It's a huge difference! Thanks to the whole community for making this happen! The first pull request come at the [beginning of February](https://github.com/napalm-automation/napalm/pull/154) and caused David Barroso excitement but also a heaadache reviewing a pretty big PR :)

#### Salt

Despite of years of experience working with this Salt and several thousands of serves managed, historically this product did not offer any capability for network devices. But from  [2015.8](https://docs.saltstack.com/en/latest/topics/releases/2015.8.0.html#proxy-minion-enhancements) on, SaltStack developers have introduced the [proxy minion feature](https://docs.saltstack.com/en/latest/topics/proxyminion/index.html). The first pull request to introduce NAPALM was [at the end of February](https://github.com/saltstack/salt/pull/31431).

## Installation

For complete installation instructions, please proceed to the [salt-users forums](https://groups.google.com/forum/#!msg/salt-users/4Fvl_yonJ9Y/8x4qrk9wAwAJ).

## Usage

As presented in &para; *How to use?* from the [first blog post](https://mirceaulinic.net/2016-11-17-network-orchestration-with-salt-and-napalm/), when the installation is complete, you only need a file containing the device connection details, as described in the [NAPALM proxy documentation](https://docs.saltstack.com/en/develop/ref/proxy/all/salt.proxy.napalm.html).

Example:

```yaml
proxy:
    proxytype: napalm
    driver: junos
    host: edge01.bjm01
    username: my_username
    passwd: my_password
    optional_args:
        port: 12201
        config_format: set
```

Way to go! From here on, imagination is the only limit!
