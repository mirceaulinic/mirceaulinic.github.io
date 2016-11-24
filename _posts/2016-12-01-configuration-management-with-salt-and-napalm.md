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

Example - set a NTP server on all Arista devices (they are selecting using the grains we have discussed about last time):

```bash
salt -G 'vendor:arista' net.load_config text='ntp server 172.17.17.1'
```

Which will return the following output for each device matched:

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
         sflow sample 16384
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

The result for each device contains the following keys:

* ```already_configured``` which says if there were changes to be applied.
* ```comment``` contains human-readable explanation in case anything failed, or messages from the system.
* ```diff``` is the configuration diff.
* ```result``` is another flag the says if the action was executed successfully.

Both flags ```result``` and ```already_configured``` are very useful when the output is reused in other modules (as they are just Python objects).

Looking at the output above, ```edge01.bjm01``` and ```edge01.pos01``` are Arista switches. ```edge01.pos01``` did not require any changes required thus the flag ```already_configured``` is set as ```True```, whilst for ```edge01.bjm01``` it is displayed the configuration diff.

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

Which applies the config, retrieves the diff and discards - in this case the field ```comment``` will notify the user about that.

For more complex configuration changes, the static configuration can be stored in a file and call specifying the **absolute** path.

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
$ sudo salt -G 'vendor:arista' net.load_config /home/mircea/arista_ntp_servers.cfg test=True
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

In order to replace the config, you will need to set the argument ```replace```: ```$ sudo salt edge01.bjm01 net.load_config /home/mircea/arista_complete_config.cfg replace=True```.

The method presented above is not quite optimal as in the configuration of a network device there can be many variations (IP addresses etc.). For more complex computations, the following method is preferred over.

## Configuration templates

Using one of the [supported templating engines](https://docs.saltstack.com/en/develop/ref/renderers/all/index.html) we can easier control the configuration and have it consistent across the network.
In this tutorial I will be working only with Jinja templates, although otther users may prefer [cheetah](https://pythonhosted.org/Cheetah/) or [mako](http://www.makotemplates.org/) etc.

The command used is [net.load_template](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_network.html#salt.modules.napalm_network.load_template). It works very similar to the previous command (by default will load merge, commit etc.) - to change this behaviours one can use again ```test```, ```replace``` arguments. Examples:

* load template defined in line:

```bash
$ sudo salt -G 'vendor:arista' net.load_template set_hostname template_source='hostname {{ host_name }}' host_name='arista.lab' test=True
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

The examples above are very very simple, meant to provide the very first steps. Moving forward, let's define a more complex template which is vendor agnostic. We can achieve this using the grains, as they are dymanic and don't require us to write manually anything.

**/home/mircea/example.jinja**
```jinja
{% set router_vendor = grains.get('vendor') -%}
{% set hostname = pillar.get('proxy', {}).get('host') -%}
{% if router_vendor|lower == 'juniper' %}
system {
    host-name {{hostname}}.lab;
}
{% elif router_vendor|lower in ['cisco', 'arista'] %}
hostname {{hostname}}.lab
{% endif %}
```

And now we can run agains all devices, no matter the vendor:

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

Which proves that no matter the vendor, running the command above changed the hostname accordingly and displayed the diff. And we did not provde any data - Salt knows that edge01.bjm01 is an Arista and edge01.flw01 is a Juniper router! Following the model above, you can go further and define more complex templates according to your specific needs.

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

Although that minimalist example used a template under an arbitrary path, this is not quite a good practice! In order to keep your system flexible to various environment changes (e.g. move the config files on a different server etc.), the recommended way is to define the templates under the Salt envrionment. Where? Under the directory specified as ```file_roots``` in the [master config file](https://github.com/napalm-automation/napalm-salt/blob/master/master) - default is ```/etc/salt/states/```.

So let's consider we are placing now our previous example under the ```file_roots```, thus ```/etc/salt/states/example.jinja```. From now on we can run:

```bash
$ sudo salt edge01.flw01 net.load_template salt://example.jinja debug=True
```

The result is the same, just that the file is specified using the ```salt://``` prefix which tells salt to look for that template under the ```file_roots```. Changing the configuration of the master or migrating to a different server will not require you to change the command format (which is a big plus when the command is scheduled!).

But wait: there's more! You can also render remote templates! Let's consider the following [NAPALM template for NTP peers](https://github.com/napalm-automation/napalm-ios/blob/develop/napalm_ios/templates/set_ntp_peers.j2) on IOS. Shrinking the URL to [http://bit.ly/2gKOj20](http://bit.ly/2gKOj20) we can not use it to load a config on a IOS device using this remote template:

```bash
$ sudo salt -G 'os:ios' net.load_template http://bit.ly/2gKOj20 debug=True
```

Other options for remote templates can be specified using ```https://``` and ```ftp://```.

### Advanced templating

Yet another benefit of Salt is that you can use inside the template the output of any of the available [execution modules](https://docs.saltstack.com/en/develop/ref/modules/all/index.html). As one can easily notice, there are hundreds. You can for example extract some information very easily using the [postgres module](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.postgres.html#salt.modules.postgres.psql_query) from a Postgres databse and based on that generate the config etc. The possibilities are literally unlimited!

Inside the template, the result of the query execution is one single line:

```jinja
{% query_results = salt['postgres.psql_query']("SELECT * FROM net.ip_addresses", db_user, db_host, db_port, db_password) -%}
```

And then use the ```query_results``` as needed!
Exacly in the same manner, we can use the network-related NAPALM modules.

For example: say we need to generate configuration to have static ARP entries, based on the existing ARP table. The following short template does the job, using the [net.arp](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_network.html#salt.modules.napalm_network.arp) function:

**/etc/salt/states/arp_example.jinja**:

```jinja
{% set arp_output = salt['net.arp']() -%}
{% set arp_table = arp_output['out'] -%}
interfaces {
  {% for arp_entry in arp_table -%}
    {{ arp_entry['interface'] }} {
      family inet {
        address {{ arp_entry['ip']}} {
          arp {{ arp_entry['ip'] }} mac {{ arp_entry['mac'] }};
        }
      }
    }
  {% endfor -%}
}
```

Running:

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
        interfaces {
          ae1.1234 {
            family inet {
              address 10.10.1.1 {
                arp 10.10.1.1 mac 9C:8E:99:15:13:B3;
              }
            }
          }
          xe-0/0/0.0 {
            family inet {
              address 10.10.2.2 {
                arp 10.10.2.2 mac 0C:86:10:F6:7C:A6;
              }
            }
          }
        }
    result:
        True
```

Again, we did not define any static data at all. The whole information was dynamically collected as the result of ```net.arp```, as well as it could be from [route.show](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.napalm_route.html#salt.modules.napalm_route.show) or [redis.hgetall](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.redismod.html#salt.modules.redismod.hgetall), or even generate config based on [nagios](https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.nagios.html#salt.modules.nagios.run) data.


## Conclusion

Inittially I had the intention to discuss today also about the [states](https://docs.saltstack.com/en/develop/ref/states/all/index.html) but it turns out I should leave it for the next time. We have seen how everything comes glued together in Salt and how the information from different processes can be used in order to generate configurations with ease, without reuiring to manually update data files.