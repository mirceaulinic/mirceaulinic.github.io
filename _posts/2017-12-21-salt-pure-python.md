---
layout: post
title: Slaying Salt using pure Python
subtitle: The overlooked side of Salt and few best practices
bigimg: /img/python.jpg
---

One of the (many) things I like about Salt is that it doesn't have an
obscure language of its own: as I always like to say, to start automating
everything you need to know is YAML and Jinja, 3 rules each. For example,
when you need a simple iteration you don't need to check the documentation
and see "what's that specific instruction that iterates through a list", but
just a straight and simple Jinja loop, i.e., ``{%- for element in list %}``.
However, there are particular cases where Jinja itself is not enough, or
it simply can become too complex. When I need to deal with a complex task,
I sometimes feel that "I'd better write this in Python than Jinja".

As we all know already, Salt is an extremely flexible framework, due to its
simple internal architecture: a dense core that allows pluggable interfaces
to be added. For instance, if you look at the [official repository on
GitHub](https://github.com/saltstack/salt/tree/develop/salt), you will notice
a pretty long list of directories (and many others can be added): ``acl``,
``auth``, ``engines``, ``beacons``, ``netapi``, ``states``, and so on; they all
are just pluggable interfaces for different subsystems that have been added
as Salt has grown in capabilities.

The pure Python Renderer
------------------------

One of these pluggable interfaces is called *Renderer*: this is a subsystem
that allows the low level input-output. For Salt it doesn't matter if your
file is structured as YAML, or JSON, or TOML etc. - the data is represented
as Python object: you're working with data, not with chunks of text!

Thanks to this intelligent approach, adding a pure Python renderer (a few years
back - 6 years now) was certainly the most natural thing to do. This is why
you are today able to write - without exaggeration - everything in pure Python.
But don't just take my word, please bear with me and I will prove.

# The pure Python SLS

If you are new to Salt, SLS (SaLtStack) is the file format used by Salt. By
default, SLS is Jinja + YAML (Jinja is rendered first), then the resulting
YAML is translated into a Python object and loaded in memory. Let's consider
the following SLS example:

```yaml
ip_addresses:
  - 10.10.10.0
  - 10.10.10.1
  - 10.10.10.2
  - 10.10.10.3
  - 10.10.10.4
```

In this simple example, we define a list of IP addresses, as plain YAML. But
what happens if we have a longer list of addresses and we want to avoid typing
each and every one manually? As previously said, SLS is by default a smart
combination of YAML and Jinja, so you can rewrite the equivalent SLS, as
follows:

```jinja
ip_addresses:
{%- for i in range(5) %}
  - 10.10.10.{{ i }}
{%- endfor %}
```

Even though for this particular example one could argue that it doesn't shrink
the size massively, it certainly does when dealing with a huge amount of data!

Remember: you can apply the exact same logic in *any* SLS file, whether it is
a Pillar, a State, a Reactor, or the ``top.sls`` file, and so on.

Before going forward, I would like to clarify that both Jinja and YAML belong
to that Renderer interface I reminded before. In other words, the SLS is using
by default these two Renderers.

You are able to choose any combination of Renderers that works for you, and you
can do this very easily, just adding a shebang (``#!``) at the top of the file
and name the Renderers you'd like to use, separated by pipe (``|``).
With these said, the default header is ``#!jinja|yaml``.

The shebang required to select the Python renderer is ``!#py``. The only
constraint is that you need a function name ``run`` that returns the data you
need. For instance, the equivalent SLS for the examples above would be the
following:

``/etc/salt/pillar/ip_addresses_py.sls``
```python
#!py

def run():
    return {
        'ip_addresses': ['10.10.10.{}'.format(i) for i in range(5)]
    }
```

And this is as simple as it looks like: we are writing pure Python that can be
used, for example, as input data for our system. So let's do that: save this
content to ``/etc/salt/pillar/ip_addresses_py.sls`` and referencing this file
in the Pillar top file (``/etc/salt/pillar/top.sls`` - as configured on the
Master, in the [pillar_roots](https://docs.saltstack.com/en/latest/ref/configuration/master.html#pillar-roots)),
in such a way that any Minion can read the contents (due to the ``*`` - see
[the top file documentation](https://docs.saltstack.com/en/latest/ref/states/top.html)
for a more details):

``/etc/salt/pillar/top.sls``
```yaml
base:
  '*':
    - ip_addresses_py
```

After refreshing the Pillar (using the ``saltutil.refresh_pillar`` execution
function), we can verify that indeed the data is available:

```bash
$ sudo salt 'minion1' saltutil.refresh_pillar
minion1:
    True
$ sudo salt 'minion1' pillar.get ip_addresses
minion1:
  - 10.10.10.0
  - 10.10.10.1
  - 10.10.10.2
  - 10.10.10.3
  - 10.10.10.4
```

But wait: I defined the Pillar top file as default SLS, purely YAML. Even
though in this trivial example it is overkill, there are good production cases
when the top file can be equally made as dynamic as needed, hence we have the
possibility to dynamically bind Pillars to Minions (or the SLS States, for the
State top file):

``/etc/salt/pillar/top.sls``
```python
#!py

def run():
    return {
        'base': {
            '*': [ 'ip_addresses_py' ]
        }
    }
```

And with this we have confirmed that we are able to introduce data into the
system using *only* Python. Although the examples I provided so far are trivial,
they can be extended to more complex implementations, as much as required to
solve the problem.

An example that I always like to give is loading Pillar data from external
systems, say from an HTTP API accessible at https://example.com/api.json (that
provides data formatted as JSON):

``/etc/salt/pillar/example.sls``
```python
#!py

import salt.utils.http

def run():
    ret = salt.utils.http.query('https://example.com/api.json', decode=True)
    return ret['dict']
```

With this 5 liner SLS using the Python renderer we can directly introduce data
into Salt. Of course, there are security and other considerations you need to
evaluate before an implementation like that. Besides that - when dealing with
very complex problems you'll need to look at the problem from another angle
and you should always consider using the [External Pillar](https://docs.saltstack.com/en/latest/topics/development/external_pillars.html)
or [External Tops](https://docs.saltstack.com/en/latest/topics/master_tops/)
systems, as they are another nice way to deal with input data. Moreover, they
offer the flexibility to be loaded before or after the regular Pillars (using
the [``ext_pillar_first`` Master configuration
option](https://docs.saltstack.com/en/latest/ref/configuration/master.html#ext-pillar-first).
There isn't a general recommendation: each particular case must be analysed
individually. And writing an extension module in your own environment for the
External Pillar subsystem is [very
easy](https://docs.saltstack.com/en/latest/ref/configuration/master.html#ext-pillar-first).

Writing executing modules
-------------------------

From experience, I can tell that the execution modules are by far the most

# Invoking execution modules inside template

Conclusions
-----------


