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
are easy enough in theory (actually in practice too), when releasing code often
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
To be able to use this module in older Salt releases, simply copy this file into
one of your ``file_roots`` directory. For example, if one of the ``file_roots``
directory is ``/etc/salt/extmods``, then you only have to execute, e.g.,

```bash
$ cd /etc/salt/extmods
$ mkdir _modules && cd _modules/
$ wget https://raw.githubusercontent.com/saltstack/salt/2019.2/salt/modules/napalm_mod.py
```

*Remember* to always use ``wget`` with the _raw_ link from GitHub, otherwise
you'll download an HTML file instead. :-)

Writing Execution Modules is easy
---------------------------------

