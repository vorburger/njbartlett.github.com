---
layout: post
title: Bndtools 2.0 Released
tags: java osgi bndtools
date: 2013-02-11 12:00:00
---

After a long and grueling process, I'm pleased (and considerably relieved) to announce that Bndtools 2.0 has been released!

There are many, many improvements in this release, which I will only summarise below; see the [release notes](http://bndtools.org/whatsnew2-0-0.html) for more details. All together I believe that Bndtools is now the foremost tool for developing OSGi applications, and continues our mission to make OSGi development *easier* and *more productive* than traditional Java development.

* Support for OSGi Release 5 Resolver and Repository specifications;
* Export run descriptors as standalone executables;
* Baselining (build errors for incorrectly versioned bundles);
* Enhanced Semantic Versioning, using annotations for Consumer and Provider roles;
* Exported Package Decorations;
* Improved Incremental Builder;
* Support for Apache ACE;
* Lots and lots of bug fixes and performance improvements (e.g. `Conditional-Package` processing was sped up over 100 times!).

Version 2.0 has been such a huge release because of the sheer amount of changes introduced in both bnd and in Bndtools. In the future we expect to make incremental releases much more frequently, and this includes a commitment not to change the structure of Bndtools workspaces (or if they need to change, to provide tools that make the migration transparent).

Bndtools 2.0 is available now on the Eclipse Marketplace. For other installation options refer to the [installation guide](http://bndtools.org/installation.html).