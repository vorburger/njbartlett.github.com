---
layout: post
title: Announcing the bnd Maven Plugin
summary: 
tags: osgi bnd maven
date: 2015-03-27 12:00:00
---


I am pleased to announce the availability of a new Maven plugin for building
OSGi bundles, based on bnd. The plugin is called `bnd-maven-plugin` and it will
be developed as part of the wider bnd and Bndtools projects.

The plugin has been kindly donated by one of Paremus’ customers -- a very well
known company whose identity we expect to be able to confirm in due course --
under the terms of the Apache License version 2.

You might very well wonder: why did we build a new Maven plugin for OSGi
development? Most OSGi developers who use Maven today are using the
`maven-bundle-plugin` from Apache Felix. That plugin is mature and stable but
it does have a number of issues that our customer wanted resolved.

The first major issue with maven-bundle-plugin is that it replaces Maven's
default plugin for JAR generation (the `maven-jar-plugin`). This means that it
must use a different packaging type -- "bundle" instead of "jar" -- and must
use the `extensions=true` setting to override Maven's default lifecycle.  As a
result, the output artifacts are harder to consume in ordinary non-OSGi
projects. Therefore, many Maven developers either don't bother to release OSGi
bundles at all because of the hassle, or they release both a "plain JAR" *and*
a separate OSGi bundle. This is deeply frustrating because OSGi bundles already
are plain JARs!... they can be used unalterered in non-OSGi applications.

The new bnd-maven-plugin instead hooks into Maven's `process-classes` phase. It
prepares all of the generated resources, such as MANIFEST.MF, Declarative
Services descriptors, Metatype XML files and so on, under the `target/classes`
directory, where they will be automatically included by the standard
maven-jar-plugin (with one caveat discussed below).

The second issue is that maven-bundle-plugin has some questionable features and
design choices. For example it automatically exports **all** packages that do not
contain the substrings "impl" or "internal". The motivation was
understandable at the time -- it is closer to normal Java with all packages
being implicitly exported -- but this is really just wrong from a modularity point
of view, and goes against OSGi best practices. The bnd-maven-plugin instead
includes all packages from the project in the bundle but exports nothing by
default. The developer must choose explicitly which packages are exported.

The third issue with maven-bundle-plugin is that it is difficult to use in an
incremental build environment. In Bndtools we have had literally years of
discussions about how to best accommodate Maven users while still supporting
the rapid development cycle that Bndtools users love. The new plugin will make
this easier: for example after the `process-classes` phase has completed, the
content of the `target/classes` directory is already a valid OSGi bundle,
albeit in folder form rather than a JAR file. Also the new plugin takes its bnd
instructions from the bnd.bnd file in each project, rather than from a funky
XML encoding of those instructions deep inside the POM.

Finally, we believe that by delivering a Maven plugin directly from the bnd
project, we can more quickly take advantage of new features in bnd, and develop
features in bnd that directly benefit Maven users. Note however that the Gradle
and Ant integrations of bnd will still be supported: we want everybody to use
bnd to build their OSGi bundles, irrespective of IDE and build system.

So much for motivation, let's look at some of the technical details.

The plugin has been released as version 2.4.1 because there was a previous
experimental plugin with the same name. That experiment was not a success, and
we abandoned it some time ago -- the new plugin is a complete rewrite. The
version of the plugin will track the version of bnd itself, so 2.4.1 of the
plugin builds against 2.4.1 bnd, which is released and available from Maven
Central.  From now on we will work on version 3.0, which will track version 3.0
of bnd.

The plugin generates the JAR manifest into
`target/classes/META-INF/MANIFEST.MF`, but due to a limitation of the
maven-jar-plugin this file is ignored and replaced by an empty, non-OSGi
manifest. We are preparing a patch against maven-jar-plugin to modify this
behaviour and hope that it will be accepted, but in the meantime it is
necessary to set the following configuration one time in your parent POM:

    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <configuration>
            <archive>
                <manifestFile>${project.build.outputDirectory}/META-INF/MANIFEST.MF</manifestFile>
            </archive>
        </configuration>
    </plugin>


**The plugin is available now [from Maven
Central](https://oss.sonatype.org/#nexus-search;quick~bnd-maven-plugin) or [jCenter](https://bintray.com/bnd/bnd/bnd-maven-plugin/view)**.
It is not yet thoroughly documented, although bnd itself does have very good
[documentation](http://bnd.bndtools.org/). All options from bnd automatically
work in bnd-maven-plugin, with the exception of "sub bundles" (`-sub`) since
Maven projects are meant to produce exactly one output artifact per project.

To add the plugin to your project, insert the following into your POM:

    <plugin>
        <groupId>biz.aQute.bnd</groupId>
        <artifactId>bnd-maven-plugin</artifactId>
        <version>2.4.1</version>

        <executions>
            <execution>
                <goals>
                    <goal>bnd-process</goal>
                </goals>
            </execution>
        </executions>
    </plugin>

We have [example projects](https://github.com/bndtools/bnd/tree/bnd-maven-plugin-2.5/bnd-maven-plugin-examples) to get started. Questions, problems or offers of help can be can be directed to the [bnd/Bndtools mailing list](https://groups.google.com/forum/#!forum/bndtools-users).
