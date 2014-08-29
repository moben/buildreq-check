buildreq-check
==============

A script to check for superfluous BuildRequires in rpm specfiles.

This script takes a .src.rpm and builds it multiple times using mock, gradually leaving out BuildRequires of the original package.
It then compares the resulting rpms with the original. If they are identical, the corresponding BuildRequires is marked as possibly unneeded. It only builds the .src.rpm about count(BuildRequires) times.

Run

    buildreq-check --help

for more information.


Limitations
===========

Before doing any actual work, buildreq-check does two builds of the .src.rpm - if those aren't identical it aborts, because it wouldn't be able to compare the later builds anyway.

Often times, a BuildRequires is used during tests but when it is missing, the tests are just skipped and the build doesn't fail. These dependencies might show up as false positives.

Some files are very unlikely to be built reproducibly (e.g. pythons .pyc and .pyo files) there are some quirks in buildreq-check that try to deal with those, but the intention is to keep those to a minimum and instead fix things upstream/downstream.

It does parallelize badly on one machine, because it builds lots of mock chroots which takes the global yum lock.


TODO
====

  - Add an optional argument to only check one BuildRequire
  - Automatically file bugs? (add a blacklist first...)
  - Run periodically on base/all packages


Status in Fedora
================

With a few patches (see below) about 90% of the packages in the tentative group of Base-packages build reproducibly - meaning that each build has the same result and this tool can be used on them.

Ideally, we would do all builds with the same packages.
There is a yum bug at the moment that prevents that though:
On a F21 system, some packages have to be patched (see below) and put in a local repo.
mock --offline passes --cacheonly to yum, which prevents it from downloading new rpms.
However, yum doesn't cache the rpms from the local repo because they use a 'file://' url, so it can't find them.
There is a flag in the script that allows offline mode to be enabled, but it is currently disabled.
Two possible workarounds:
  - Set the cache expiry up and disable background cache update services
  - Create a local repo with _all_ needed rpms and use that exclusively.


Patches
=======

Upstream
--------

  - [autogen](http://sourceforge.net/p/autogen/bugs/163/): don't add timestamps to generated files
  - [cython](https://github.com/cython/cython/pull/310): don't add timestamps to generated files

Submitted
---------
  - [rpmbuild](http://lists.rpm.org/pipermail/rpm-maint/2014-August/003725.html):
    - export build-time to rpmbuild environment
    - use that to touch .py files, resulting in reproducible .pyc and .pyo files

    could also be used for other purposes, such as:
    - repacking .jar files (which we already do, but with non-sensical timestamps)
    - fixing timestamps where upstream is uncooperative/dead (might be simpler than carrying patches)
  - [python-wheel](https://bitbucket.org/pypa/wheel/pull-request/47): make order of lines in py-dist metadata deterministic after
    pythons hash-randomization was enabled by default
  - [python-setuptools](https://bitbucket.org/pypa/setuptools/pull-request/79): make order of lines in py-dist metadata deterministic after
    pythons hash-randomization was enabled by default

Rejected/Dead
-------------
  - [javadoc](http://mail.openjdk.java.net/pipermail/javadoc-dev/2014-July/000138.html): don't add timestamps to generated files
  - [epydoc](http://sourceforge.net/p/epydoc/patches/26/): don't add timestamps to generated files

Those can probably be worked aroud with the rpmbuild patch or carried in the respective packages if the maintainers are willing.
    
Fedora
------
  - submitted patches/backports for epydoc, rpmbuild, python-setuptools, cython
  - emacs: applied upstream patch to drop timestamps from .elc files
  - enabled deterministic archives in binutils for F22 (fixed timestamps/ permissions inside .a files)
  - filed bugs/fixed some low-hanging fruit discovered by this tool: policycoreutils, autoconf213, gobject-introspection

