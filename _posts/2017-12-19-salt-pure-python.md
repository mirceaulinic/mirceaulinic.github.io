---
layout: post
title: Python development for infrastructure management using Salt
subtitle: The overlooked side of Salt and some best practices
bigimg: /img/flexwing.jpg
---

One of the (many) things I like about Salt is that it doesn't have an
obscure language of its own: as I always like to say, to start automating
all you need to know is YAML and Jinja, 3 rules each. For example, when you need
a simple iteration you don't need to check the documentation and see _"what's
that specific instruction that iterates through a list"_, but just a simple and
straight Jinja loop, i.e., {% raw %}``{%- for element in list %} .. {%- endfor %}``{% endraw %}.
However, there are particular cases where Jinja itself is not enough either, or
it simply can become too complex and unreadable. When I need to deal with a
complex task, I sometimes feel that _"I'd better write this in Python than
Jinja (or a combination of both)"_.

As we know already, Salt is a very flexible framework, due to its
simple internal architecture: a dense core that allows pluggable interfaces
to be added. For instance, if you look at the [official repository on
GitHub](https://github.com/saltstack/salt/tree/develop/salt), you will notice
a pretty long list of directories (and many others can be added): ``acl``,
``auth``, ``engines``, ``beacons``, ``netapi``, ``states``, and so on; they all
are just pluggable interfaces for different subsystems that have been added
as Salt has grown in capabilities.

# The pure Python Renderer

One of these pluggable interfaces is called *Renderer*: this is a subsystem
that facilitates the low level input-output interaction. For Salt it doesn't
matter if your file is structured as YAML, or JSON, or TOML etc. - the data is
represented as Python object: you're working with data, not with chunks of text!

Thanks to this intelligent approach, adding a pure Python renderer (about 6
years ago) was certainly the most natural thing to do. This is why you are able
today to write - without exaggeration - everything in pure Python.
But don't just take my word, bear with me and I will prove.

## Python Pillars

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

{% raw %}
```jinja
ip_addresses:
{%- for i in range(5) %}
  - 10.10.10.{{ i }}
{%- endfor %}
```
{% endraw %}

Even though in this particular example one could argue that it doesn't shrink
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

The shebang required to select the Python renderer is ``#!py``. The only
constraint is that you need a function named ``run`` that returns the data you
need. For instance, the equivalent SLS for the examples above would be the
following:

``/etc/salt/pillar/ip_addresses.sls``
```python
#!py

def run():
    return {
        'ip_addresses': ['10.10.10.{}'.format(i) for i in range(5)]
    }
```

And this is as simple as it looks like: we are writing pure Python that can be
used, for example, as input data for our system. So let's do that: save this
content to ``/etc/salt/pillar/ip_addresses.sls`` and referencing this file
in the Pillar top file (``/etc/salt/pillar/top.sls`` - as configured on the
Master, in the [pillar_roots](https://docs.saltstack.com/en/latest/ref/configuration/master.html#pillar-roots)),
in such a way that any Minion can read the contents (due to the ``*`` - see
[the top file documentation](https://docs.saltstack.com/en/latest/ref/states/top.html)
for a more details):

``/etc/salt/pillar/top.sls``
```yaml
base:
  '*':
    - ip_addresses
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
possibility to dynamically bind Pillars to Minions (or Formulas, for the
State top file):

``/etc/salt/pillar/top.sls``
```python
#!py

def run():
    return {
        'base': {
            '*': [ 'ip_addresses' ]
        }
    }
```

And with this we have confirmed that we are able to introduce data into the
system using *only* Python. Although the examples I provided so far are trivial,
they can be extended to more complex implementations, as much as required to
solve the problem.

An example that I always like to give is loading Pillar data from external
systems, say from an HTTP API accessible at _https://example.com/api.json_ (that
provides data formatted as JSON):

``/etc/salt/pillar/example.sls``
```python
#!py

import salt.utils.http

def run():
    ret = salt.utils.http.query('https://example.com/api.json', decode=True)
    return ret['dict']
```

With this 5 liner SLS using the Python renderer, we can directly introduce data
into Salt from a remote endpoint. Of course, there are security and other
considerations you need to evaluate before an implementation like that. Besides
that - when dealing with very complex problems you'll need to look at the
problem from more than one angle and you should always consider using the
[External Pillar](https://docs.saltstack.com/en/latest/topics/development/external_pillars.html)
or [External Tops](https://docs.saltstack.com/en/latest/topics/master_tops/)
systems, as they are another nice way to deal with input data. Moreover, they
offer the flexibility to be loaded before or after the regular Pillars (using
the [``ext_pillar_first`` Master configuration
option](https://docs.saltstack.com/en/latest/ref/configuration/master.html#ext-pillar-first)).
There isn't a general recommendation: each particular case must be analysed
individually. Writing an extension module in your own environment for the
External Pillar subsystem is also [very
easy](https://docs.saltstack.com/en/latest/ref/configuration/master.html#ext-pillar-first).

## Python SLS Formulas

In a similar way we can write our SLS Formulas purely in Python. For example,
the following State SLS (taken from the [napalm-ntp-formula](https://github.com/saltstack-formulas/napalm-ntp-formula)):

{% raw %}
```yaml
oc_ntp_netconfig:
  netconfig.managed:
    - template_name: salt://ntp/templates/init.jinja
    - {{ salt.pillar.get('openconfig-system') }}
```
{% endraw %}

Can be rewritten, using the ``py`` Renderer:

```python
#!py

def run():
    return {
        'oc_ntp_netconfig': {
            'netconfig.managed': [
                {'template_name': 'salt://ntp/templates/init.jinja'},
                __salt__['pillar.get']('openconfig-system')
            ]
        }
    }
```

Again, in this particular example, the Python Renderer doesn't bring more
benefits compared to its default behaviour, but it was a good occasion to see
introduce the ``__salt__`` dunder which is the hash with all the execution
functions available.
Writing Python Formulas proves very useful when our states require very complex
decisions.

I prefer (and recommend) however moving the complexity into an execution
function, as detailed in the _Writing Executing Modules_ section.

## Python templates

Yes, you read that right, and it is not overzealous. Again, there are cases
when Jinja or another templating engine are not enough. For example, I use this
when I need to generate text containing unicode characters, as Jinja is really
bad at doing that, or overly complicated, while in Python it is as simple as
adding ``# -*- coding: utf-8 -*-`` at the top of the file.

If you think about it, at the end of the day, a template engine only returns a
chunk of plain text given a set of input variables. Achieving that using Python
is close to trivial. There is another element specific to the Salt ``py``
Renderer you need to be aware of: where to find the input variables. Salt
injects a global variable named ``context`` that is a dictionary containing the
variables you are sending to the template.

Consider the following template (the file extension doesn't actually
matter, but it's good to keep it consistent so you and your text editor know
the file format):

``/etc/salt/templates/example.py``
```python
#!py

def run():
    length = context['ntp_peers_count']
    lines = []
    for i in range(length):
        lines.append('set system ntp peer 10.10.10.{}'.format(i))
    return '\n'.join(lines)
```

This Python template can be used as any other: we only need to tell Salt
to generate the text via the ``py`` engine. We can verify and load the generated
content, from the CLI, using the ``net.load_template`` execution function
(``minion1`` is a Juniper network device):

```bash
$ sudo salt 'minion1' net.load_template salt://templates/example.py \
debug=True test=True template_engine=py ntp_peers_count=2
minion1:
    ----------
    already_configured:
        False
    comment:
        Configuration discarded.
    diff:
        [edit system ntp]
        +    peer 10.10.10.0;
        +    peer 10.10.10.1;
    loaded_config:
        set system ntp peer 10.10.10.0
        set system ntp peer 10.10.10.1
    result:
        True
```

Or via the State system:

``/etc/salt/states/example.sls``
```yaml
/tmp/ntp_peers.cfg:
  file.managed:
    - source: salt://templates/example.py
    - template: py
    - context:
        ntp_peers_count: 2
```

When executing the ``example.sls`` Formula, it will generate the ``/tmp/ntp_peers.cfg``
text file, processing the ``salt://templates/example.py`` template (remember
that ``salt://`` points to the location of the Salt fileserver, which is
``/etc/salt`` in this case, as configured via ``file_roots``) through
the ``py`` interface. Executing ``$ sudo salt 'minion1' state.sls example``, it
will result the following content:

```bash
$ cat /tmp/ntp_peers.cfg
set system ntp peer 10.10.10.0
set system ntp peer 10.10.10.1
```

# Writing Executing Modules

It is not a secret that the Execution Modules are the most flexible subsystem
in Salt, as they allow you to reuse code, thanks to the fact that they are
available in various other subsystems, including: Renderers (and, implicitly,
templates), State modules, Engines, Returners, Beacons, and so on. Basically
once you wrote an Execution Module it is available immediately in the named
parts of Salt.

All you need to write an Execution Module is very basic knowledge of Python,
and read the [``file_roots``](https://docs.saltstack.com/en/latest/ref/configuration/master.html#file-roots)
documentation to understand how the Salt filesystem works and where to save the
files. Suppose we have the following ``file_roots`` configuration:

```yaml
file_roots:
  base:
    - /etc/salt
```

Then just save a file in a directory called ``_modules`` under one of the
paths referenced in the ``file_roots``, e.g., ``/etc/salt/_modules``:

``/etc/salt/_modules/ip_addresses.py``
```python
def generate(length=5):
    return ['10.10.10.{}'.format(i) for i in range(length)]
```

##### Tip

> There is a massive arsenal of helper functions that you can re-use. They are
> found in the [``utils``](https://github.com/saltstack/salt/tree/develop/salt/utils)
> directory. Take a few moments to skim through this directory and its files. Don't
> worry, from experience I can tell that it will take you months or years to
> know where to look for the exact function you need, in order to avoid reinventing wheels.
> Been there, done that, got the "wheel reinventor" t-shirt. :-)
>
> Moreover, remember that you can invoke execution functions from other execution
> function, as described below.

To let Salt know that there is another Execution Module to load, you must
run ``saltutil.sync_modules``, and simply execute the newly defined function
(the syntax being ``<module name>.<function name>``; in our case the module
name is the name of the file save, ``ip_addresses``, and the name of the
function is ``generate``):

```bash
$ sudo salt 'minion1' saltutil.sync_modules
minion1:
    - modules.ip_addresses
$ sudo salt 'minion1' ip_addresses.generate
minion1:
    - 10.10.10.0
    - 10.10.10.1
    - 10.10.10.2
    - 10.10.10.3
    - 10.10.10.4
$ sudo salt 'minion1' ip_addresses.generate length=1
minion1:
    - 10.10.10.0
```

Note in the last example the key-value argument ``length`` is passed from the
CLI to the ``generate`` function, with the name preserved as we defined in the
Python module.

##### Note

> By default, the name of the Execution Module is simply the name of the
> Python module (file).
> To use a different name instead, you can use the ``__virtualname__`` dunder.
> This is a beautiful way to overload the name depending on special
> circumstances, which is a unique capability of Salt.
> For more details, please refer to [this page](https://docs.saltstack.com/en/latest/ref/modules/#virtualname).

So we have a new function defined for our own environment. This can be invoked
from the command line, as we've seen, but also available in different areas,
as follows.

## Invoking Execution Modules inside template

The new ``ip_addresses.generate`` execution function can be called from any
of the templating languages supported by Salt, for example Jinja:

{% raw %}
```jinja
{%- for addr in salt.ip_addresses.generate(length=3) %}
IP Address {{ addr }}
{%- endfor %}
```
{% endraw %}

The template above would be rendered as:

```text
IP Address 10.10.10.0
IP Address 10.10.10.1
IP Address 10.10.10.2
```

## Invoking Execution Modules inside Pillar SLS

Using the ``ip_addresses.generate`` function we can rewrite the
``/etc/salt/pillar/ip_addresses.sls`` Pillar from the examples above as:

{% raw %}
```yaml
ip_addresses: {{ salt.ip_addresses.generate() }}
```
{% endraw %}

Invoking Execution Modules inside Formulas works in the exact same way.

## Invoking Execution Modules from other Salt modules

In general, we can use the ``__salt__`` dunder to execute a function from a
different Salt module. For example, we can define the following Execution
Module which will invoke the ``ip_address.generate`` function:

``/etc/salt/_modules/ixp_interfaces.py``
```python
def addresses(extension):
    default_addresses = __salt__['ip_addresses.generate'](length=100)
    extension_addresses = [
        '172.17.17.{}'.format(i) for i in range(extension)
    ]
    default_addresses.extend(extension_addresses)
    return default_addresses
```

Note that in the example above, the ``extension`` argument is no longer a
key-value and we will always need to pass a value when executing this function:

```bash
$ sudo salt 'minion1' ixp_interfaces.addresses 20
# Output omitted: a list of 120 IP Addresses
```

In a similar way, we can reuse the code from ``ip_addresses.generate`` in other
subsystems, such as Beacons, Engines, Runners, or Pillars etc.

Another bonus of doing this is that you can control easier various parameters.
For example, in the way we designed the ``generate`` function, it returns IP
addresses from the ``10.10.10.0`` network; let's suppose that at some point we
decide to generate addresses from the ``172.17.19.0`` network, we only have a
single place to make the adjustment. Moving forward, if this is very likely to
change frequently we can move the base into another key-value argument, or
in a configuration option:

- IP Network as kwarg:

``/etc/salt/_modules/ip_addresses.py``
```python
def generate(base='10.10.10', length=5):
    return [
        '{base}.{i}'.format(base=base, i=i) for i in range(length)
    ]
```

- IP Network as config option:

``/etc/salt/_modules/ip_addresses.py``
{% highlight python %}
def generate(length=5);
    base = __opts__.get('ip_addresses_base', '10.10.10')
    return [
        '{base}.{i}'.format(base=base, i=i) for i in range(length)
    ]
{% endhighlight %}

In the second approach, the ``__opts__`` dunder is the dictionary having the
Minion configuration options (read from the configuration file --
``/etc/salt/minion`` for regular Minions, or ``/etc/salt/proxy`` for Proxy
Minions), merged with the Pillar and Grains data. To propagate a change in your
system - as described above, would only imply adjusting the (Proxy) Minion
config file, e.g.,:

``/etc/salt/minion``
{% highlight yaml %}
ip_addresses_base: 192.168.1
{% endhighlight %}

##### Note

> Before defining your own configuration option, check that it's not already
> defined, to avoid eventual conflicts:
> [configuring the Salt Minion](https://docs.saltstack.com/en/latest/ref/configuration/minion.html).

Conclusions
-----------

As always in Salt there is no "best rule": Salt is very flexible and your
environment dictates what makes the most sense to you. Not only that Salt
exposes to you the power of Python, but it also behaves like Python from this
perspective and provides you the means to tackle any problem in several ways;
there are no hard constraints. This is why you *always* must evaluate and decide
what approach is the most suitable for you.

My recommendation is to try to move the complexity into the Execution Modules.
Yes, write many extension modules in your own environment (and it would also be
very nice for the community to open source what is not heavily tied to your
business logic). Simplify your complex Jinja templates by using execution
functions. Write many helpers for your team. Keep the SLS files extremely
simple. When an SLS (or a logical section of an SLS), or a template is longer
than 5-10 lines, you should start asking questions and find ways to optimize and
make your code more reusable. Though, when you cannot break the complexity
apart, or it doesn't necessarily make sense to be moved elsewhere, you are again
covered, and you can avoid complex YAML/Jinja by leveraging the power of the
regular Python Renderer (as an aside, it is my favourite interface).
