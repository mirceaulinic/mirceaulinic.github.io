---
layout: post
title: Network orchestration with Salt and NAPALM
subtitle: Part2: Configuration management
bigimg: /img/saltnet.jpg
---

In the [previous post](https://mirceaulinic.net/2016-11-17-network-orchestration-with-salt-and-napalm/) we have been introduced to the very basic command syntax and features. As previously said, Salt is much more than a configuration management system - it is a [data-drivern software](https://docs.saltstack.com/en/latest/topics/) built on a dynamic communication bus. Given the number of possibilites provided (now also for network devices), this post will be entirely dedicated configuration management.

# Configuration management

For network devices, currently there are three ways to apply configuration changes:

* on the fly
* template rendering
* states

Using Salt's ability to [schedule jobs](https://docs.saltstack.com/en/2015.8/topics/jobs/schedule.html), you can instruct the system to apply these changes at specific intervals, or using the [reactor system](https://docs.saltstack.com/en/latest/topics/reactor/) they can be triggered by certain events (e.g.: say a BGP neighbor went down, the reactor listening to the event bus will determine a BGP configuration change etc.). Those are more advanced topics to be covered later.

## On the fly configuration changes

Using the function ```load_config``` from the [net module](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_network.html#salt.modules.napalm_network.load_config) you can load static parts of configuration on the selected devices.

Example - set a NTP server on all Arista devices (they are selecting using the grains [we have discussed about last time: &para; *Grains*](https://mirceaulinic.net/2016-11-17-network-orchestration-with-salt-and-napalm/)):

```bash
salt -G 'vendor:arista' net.load_config text='ntp server 172.17.17.1'
```

Which will return the following output for every device matched:

```bash
$ sudo salt -G 'vendor:arista' net.load_config text='ntp server 172.17.17.1'
edge01.bjm01:
    ----------
    already_configured:
        False
    comment:
    diff:
        @@ -42,6 +42,7 @@
         ntp server 10.10.10.1
         ntp server 10.10.10.2
         ntp server 10.10.10.3
        +ntp server 172.17.17.1
         ntp serve all
         !
    result:
        True
edge01.pos01:
    ----------
    already_configured:
        True
    comment:
    diff:
    result:
        True
```

Again displayed nicely thanks to the [nested outputter](https://docs.saltstack.com/en/2015.8/ref/output/all/salt.output.nested.html#module-salt.output.nested). But the output is actually a Python dictionary object that can be displayed using the [raw outputter](https://docs.saltstack.com/en/2015.8/ref/output/all/salt.output.raw.html#module-salt.output.raw):

```bash
$ sudo salt --out=raw edge01.pos01 net.load_config text='ntp server 172.17.17.1'
{'edge01.pos01': {'comment': '', 'already_configured': True, 'result': True, 'diff': ''}}
```

The output contains the following keys:

* ```already_configured``` which says if there were changes to be applied.
* ```comment``` contains human-readable explanation in case anything failed, or messages from the system.
* ```diff``` is the configuration diff.
* ```result``` is another flag that says if the action was executed successfully.

Both flags ```result``` and ```already_configured``` are very useful when the output is reused in other modules (as they are just Python objects).

Looking at the output above, ```edge01.bjm01``` and ```edge01.pos01``` are Arista switches. ```edge01.pos01``` did not require any changes required thus the flag ```already_configured``` is set as ```True```, whilst for ```edge01.bjm01``` it is displayed the configuration diff. For both ```result``` was ```True``` as the command did not raise any errors.

Executing the command exactly as presented, the configuration will be committed on the device. For a dry run, you can use the ```test``` argument:

```bash
$ sudo salt edge01.bjm01 net.load_config text='ntp server 172.17.17.1' test=True
edge01.bjm01:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        @@ -42,6 +42,7 @@
         ntp server 10.10.10.1
         ntp server 10.10.10.2
         ntp server 10.10.10.3
        +ntp server 172.17.17.1
         ntp serve all
         !
    result:
        True
```

Which applies the config, retrieves the diff and discards - in this case the field ```comment``` will notify the user about that. If you need to preserve the changes without committing, the option ```commmit=False``` has to be set.

For more configuration changes, the static configuration can be stored in a file and call specifying the **absolute** path.

Say we have the following static file:

```bash
$ cat /home/mircea/arista_ntp_servers.cfg
ntp server 172.17.17.1
ntp server 172.17.17.2
ntp server 172.17.17.3
ntp server 172.17.17.4
```

And execute:

```bash
$ sudo salt edge01.bjm01 net.load_config /home/mircea/arista_ntp_servers.cfg test=True
edge01.bjm01:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        @@ -42,6 +42,10 @@
         ntp server 10.10.10.1
         ntp server 10.10.10.1
         ntp server 10.10.10.1
        +ntp server 172.17.17.1
        +ntp server 172.17.17.2
        +ntp server 172.17.17.3
        +ntp server 172.17.17.4
         ntp serve all
         !
    result:
        True
```

Which takes ~ 1 second. Running the same command against 1000 devices, the run time will be exactly the same because [Salt is parallel](https://docs.saltstack.com/en/latest/topics/#parallel-execution).

In order to replace the config, you will need to set the argument ```replace```: ```$ sudo salt edge01.bjm01 net.load_config /home/mircea/edge01_bjm01.cfg replace=True```.

The method presented above is not quite optimal as in the configuration of a network device there can be many variations (IP addresses etc.). For more complex computations, the following method is recommended.

## Configuration templates

Using one of the [supported templating engines](https://docs.saltstack.com/en/develop/ref/renderers/all/index.html) we can easier control the configuration and have it consistent across the network.
In this tutorial I will be working only with Jinja templates, although otther users may prefer [cheetah](https://pythonhosted.org/Cheetah/) or [mako](http://www.makotemplates.org/) etc.

The command used is [net.load_template](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_network.html#salt.modules.napalm_network.load_template). It works very similar to the previous command (by default will load merge, commit etc.) - to change this behaviours one can use again the arguments ```test```, ```replace```, ```commit```. Examples:

* load template defined in line -- dry-run:

```bash
$ sudo salt edge01.bjm01 net.load_template set_hostname template_source='hostname {{ host_name }}' host_name='arista.lab' test=True
edge01.bjm01:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        @@ -35,7 +35,7 @@
         logging console emergencies
         logging host 192.168.0.1
         !
        -hostname edge01.bjm01
        +hostname arista.lab
         !
         ntp source Loopback0
    result:
        True
```

* load template making use of the grains - will set the hostname based on router model:

```bash
$ sudo salt edge01.bjm01 net.load_template set_hostname template_source='hostname {{ grains.model }}.lab' test=True
edge01.bjm01:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        @@ -35,7 +35,7 @@
         logging console emergencies
         logging host 192.168.0.1
         !
        -hostname edge01.bjm01
        +hostname DCS-7280SR-48C6-M-R.lab
         !
         ntp source Loopback0
    result:
        True
```

* using data from the pillar - will append ```.lab``` at the end of the exising hostname:

```bash
$ sudo salt edge01.bjm01 net.load_template set_hostname template_source='hostname {{ pillar.proxy.host }}.lab' test=True
edge01.bjm01:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        @@ -35,7 +35,7 @@
         logging console emergencies
         logging host 192.168.0.1
         !
        -hostname edge01.bjm01
        +hostname edge01.bjm01.lab
         !
         ntp source Loopback0
    result:
        True
```

The examples above are very simple, meant to provide the very first steps. Moving forward, let's define a more complex template which is vendor agnostic. We can achieve this using the grains, as they are dymanic and don't require us to manually write anything.

**/home/mircea/example.jinja**
```jinja
{% set router_vendor = grains.vendor -%}{# get the vendor grains #}
{% set hostname = pillar.proxy.host -%}{# host specified in the pillar, under the proxy details #}
{% if router_vendor|lower == 'juniper' %}
system {
    host-name {{hostname}}.lab;
}
{% elif router_vendor|lower in ['cisco', 'arista'] %}
{# both Cisco and Arista have the same syntax for hostname #}
hostname {{hostname}}.lab
{% endif %}
```

And now we can run against all devices, no matter the vendor (notice the ```*``` selector to math any minion):

```bash
$ sudo salt '*' net.load_template /home/mircea/example.jinja
edge01.bjm01:
    ----------
    already_configured:
        False
    comment:
    diff:
        @@ -35,7 +35,7 @@
         logging console emergencies
         logging host 192.168.0.1
         !
        -hostname edge01.bjm01
        +hostname edge01.bjm01.lab
         ip name-server vrf default 8.8.8.8
         !
         ntp source Loopback0
    result:
        True
edge01.flw01:
    ----------
    already_configured:
        False
    comment:
    diff:
        [edit system]
        -  host-name edge01.flw01;
        +  host-name edge01.flw01.lab;
    result:
        True
```

Which proves that no matter the vendor, running the command above changed the hostname accordingly and displayed the diff. And we did not provde any data - Salt knows that ```edge01.bjm01``` is an Arista and ```edge01.flw01``` is a Juniper router! Following the model above, you can go further and define more complex templates according to your specific needs.

Another useful option is ```debug```, displaying the config generated after the template was rendered, in the ```loaded_config``` key:

```bash
$ sudo salt edge01.flw01 net.load_template /home/mircea/example.jinja debug=True
edge01.flw01:
    ----------
    already_configured:
        False
    comment:
    diff:
        [edit system]
        -  host-name edge01.flw01;
        +  host-name edge01.flw01.lab;
    loaded_config:
        system {
            host-name edge01.flw01.lab;
        }
    result:
        True
```

Although that minimalist example used a template under an arbitrary path, this is not quite a good practice! In order to keep your system flexible to various environment changes (e.g. move the config files on a different server etc.), the recommended way is to define the templates under the Salt environment. Where? Under the directory specified as ```file_roots``` in the [master config file](https://github.com/napalm-automation/napalm-salt/blob/master/master) - default is ```/etc/salt/states/```.

So let's consider we placed our previous example under the ```file_roots```, thus ```/etc/salt/states/example.jinja```. From now on we can run:

```bash
$ sudo salt edge01.flw01 net.load_template salt://example.jinja debug=True
```

The result is the same, just that the file is specified using the ```salt://``` prefix which tells Salt to look for that template under the ```file_roots```. Changing the configuration of the master or migrating to a different server will not require you to change the command format (which is a big plus when the command is scheduled!).

But wait: there's more! You can also render remote templates! Let's consider the following [NAPALM template for NTP peers](https://github.com/napalm-automation/napalm-ios/blob/develop/napalm_ios/templates/set_ntp_peers.j2) on IOS. Shrinking the URL to [http://bit.ly/2gKOj20](http://bit.ly/2gKOj20) we can use it now to load the config on the managed Cisco devices running IOS, using this remote template:

```bash
$ sudo salt -G 'os:ios' net.load_template http://bit.ly/2gKOj20 peers=['172.17.17.1', '172.17.17.2']
```

Other options for remote templates can be specified using ```https://``` or ```ftp://```.

### Advanced templating

Yet another benefit of Salt is that you can use inside the template the output of any of the available [execution modules](https://docs.saltstack.com/en/develop/ref/modules/all/index.html). As one can easily notice, there are hundreds. You can for example extract some information very easily using the [postgres module](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.postgres.html#salt.modules.postgres.psql_query) from a Postgres databse and based on that generate the config etc. [Read more](https://docs.saltstack.com/en/latest/topics/jinja/index.html#calling-salt-functions)

Inside the template, you can extract the data from the DB in one single line:

```jinja
{% query_results = salt['postgres.psql_query']("SELECT * FROM net.ip_addresses", db_user, db_host, db_port, db_password) -%}
```

And then use the ```query_results``` as needed!
Exacly in the same manner, we can use the network-related NAPALM modules.

#### Generate static ARP configuration, based on the existing ARP table

Say we have a very long ARP table and we need to cache it statically in the configuration of the device. The following short template does the job, using the [net.arp](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_network.html#salt.modules.napalm_network.arp) function:

**/etc/salt/states/arp_example.jinja**:

```jinja
{%- set arp_output = salt['net.arp']() -%}
{%- set arp_table = arp_output['out'] -%}

{%- for arp_entry in arp_table -%}
  {%- if grains.os|lower == 'iosxr' -%} {# if the device is a Cisco IOS-XR #}
  arp {{ arp_entry['ip'] }} {{ arp_entry['mac'] }} arpa
  {%- elif grains.vendor|lower == 'juniper' -%} {# or if the device is a Juniper #}
  set interfaces {{ arp_entry['interface'] }} family inet address {{ arp_entry['ip'] }} arp {{ arp_entry['mac'] }} mac {{ arp_entry['mac'] }}
  {%- endif %}
{%- endfor -%}
```

Running against ```edge01.flw01``` which is a Juniper device:

```bash
$ sudo salt edge01.flw01 net.load_template salt://arp_example.jinja debug=True
edge01.flw01:
    ----------
    already_configured:
        False
    comment:
    diff:
        [edit interfaces xe-0/0/0 unit 0 family inet]
        +       address 10.10.2.2/32 {
        +           arp 10.10.2.2 mac 0c:86:10:f6:7c:a6;
        +       }
        [edit interfaces ae1 unit 1234]
        +      family inet {
        +          address 10.10.1.1/32 {
        +              arp 10.10.1.1 mac 9c:8e:99:15:13:b3;
        +          }
        +      }
    loaded_config:
      set interfaces ae1.1234 family inet address 10.10.1.1/32 arp 10.10.1.1 mac 9c:8e:99:15:13:b3
      set interfaces xe-0/0/0.0 family inet address 10.10.2.2/32 arp 10.10.2.2 mac 0c:86:10:f6:7c:a6
    result:
        True
```

Which configures the static APR entries required.

#### Configure default route if not already in the table

In the device pillar (see  &para; *Proxy minion config* from the [previous post](https://mirceaulinic.net/2016-11-17-network-orchestration-with-salt-and-napalm/)) append the following line:

```yaml
default_route_nh: 1.2.3.4
```

Which defines the next-hop for the defaul route. In the pillar is the place to define static data.

In the following Jinja template we'll use this information, as well as the result of [route.show](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_route.html#salt.modules.napalm_route.show) to retrieve the operational data for the static routes to ```0.0.0.0/0```:

**/etc/salt/states/route_example.jinja**:
```jinja
{%- set route_output = salt.route.show('0.0.0.0/0', 'static') -%}
{# notice that you can also use salt['route.show'] as well as salt.route.show #}
{%- set default_route = route_output['out'] -%}

{%- if not default_route -%} {# if no default route found in the table #}
  {%- if grains.vendor|lower == 'juniper' -%}
  set routing-options static route 0.0.0.0/0 next-hop {{ pillar.default_route_nh }}
  {%- elif grains.os|lower == 'iosxr' -%}
  router static address-family ipv4 unicast 0.0.0.0/0 {{ pillar.default_route_nh }}
  {%- endif %}
{%- endif -%}
```

Executing against ```edge01.flw01``` (Juniper) and ```edge01.oua01``` (Cisco IOS-XR):

```bash
$ sudo salt -L 'edge01.flw01, edge01.oua01' net.load_template salt://route_example.jinja debug=True
edge01.flw01:
    ----------
    already_configured:
        False
    comment:
    diff:
        [edit routing-options static route 0.0.0.0/0]
        +    next-hop 1.2.3.4;
    loaded_config:
        set routing-options static route 0.0.0.0/0 next-hop 1.2.3.4
    result:
        True
edge01.oua01:
    ----------
    already_configured:
        False
    comment:
    diff:
        ---
        +++
        @@ -3497,6 +3497,7 @@
         !
         router static
          address-family ipv4 unicast
        +  0.0.0.0/0 1.2.3.4
           10.10.0.0/16 5.6.7.8
           172.17.17.0/24 Null0 tag 100
    loaded_config:
        router static address-family ipv4 unicast 0.0.0.0/0 1.2.3.4
    result:
        True
```

Installs a static route to ```0.0.0.0/0``` having as next hop ```1.2.2.4```, as there were no default static routes found in the table.

We have achieved the goals by defining less than 10 lines long templates, covering the configuration syntax for multiple vendors. Most of the data (everything, except the next-hop address) was dynamically collected from the devices, through the *grains* and the result of ```net.arp``` or ```route.show```, as well as it could be from [bgp.neighbors](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_bgp.html#salt.modules.napalm_bgp.neighbors) or [ntp.stats](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_ntp.html#salt.modules.napalm_ntp.stats) or [redis.hgetall](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.redismod.html#salt.modules.redismod.hgetall), or [nagios](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.nagios.html#salt.modules.nagios.run) or anything else. This is a genuine example of an orchestrator: configuration data depends on the operational data and vice-versa.

This is even more good news: basically you have available an army of thousands of filters waiting to be used. The difference is the syntax: instead of ```{{ 'get salted' | sha512_digest }}```, it would require writing ```{{ salt.hashutil.sha512_digest('get salted') }}``` using the [hashutil functions](https://docs.saltstack.com/en/2015.8/ref/modules/all/salt.modules.hashutil.html#salt.modules.hashutil.sha512_digest); similarly ```{{ salt.dnsutil.aaaa('www.google.com') }}``` from the [dnsutil module](https://docs.saltstack.com/en/2015.8/ref/modules/all/salt.modules.dnsutil.html#salt.modules.dnsutil.AAAA) or ```{{ salt.timezone.get_zone() }}`` to get the [timezone](https://docs.saltstack.com/en/2015.8/ref/modules/all/salt.modules.timezone.html#salt.modules.timezone.get_zone) etc.

## TBC

Inittially I had the intention introduce also the [states](https://docs.saltstack.com/en/develop/ref/states/all/index.html) but it turns out I should leave it for the next time. We have seen how everything comes glued together in Salt and how the information from different processes can be used in order to generate configurations with ease, without requiring us to manually update data files or write other external processes that collect data in order to introduce it back in the system.
