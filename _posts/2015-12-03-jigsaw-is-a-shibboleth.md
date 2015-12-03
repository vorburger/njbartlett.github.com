---
layout: post
title: Jigsaw is a Shibboleth
summary: Release Java 9 on time.
tags: osgi jigsaw bnd
date: 2015-12-03 12:00:00
---

It comes as no surprise that Oracle has [proposed to delay the release of JDK 9 for 6 months](http://mail.openjdk.java.net/pipermail/jdk9-dev/2015-December/003149.html) (but note that JSR 376 specifying the Java Platform Module System is being [delayed by a full year](http://mail.openjdk.java.net/pipermail/jpms-spec-observers/2015-December/000233.html), so Java SE 9 itself will likely be delayed by at least that long).

The reason for the delay will also be no surprise: Jigsaw is still not ready.

The crusade to develop a new module system for Java has been long and mostly fruitless. JSR 277 “The Java Module System” began over ten years ago, in June 2005… more than a year before Java 6 was released, and five years before Oracle acquired Sun Microsystems. It was meant to be delivered in Java 7 but it slipped to Java 8. When it looked like it was going to seriously delay the release of Java 8, it slipped again to Java 9. Now the same thing is happening yet again. The tongue-in-cheek prediction made by Robert Dunne is gradually coming true:

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">All I know about Java version 20 is that Project Jigsaw will be looking good and on target for version 21 <a href="http://t.co/CqHCI1nTaV">http://t.co/CqHCI1nTaV</a></p>&mdash; Robert Dunne (@rsdunne) <a href="https://twitter.com/rsdunne/status/494123207977607168">July 29, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>


It really doesn't have to be this way: Jigsaw has already achieved something extremely valuable, and Java 9 could be released on schedule with Jigsaw as it stands, in addition to the other nice features slated such as [compact Strings](http://openjdk.java.net/jeps/254) and [standard replacements for sun.misc.Unsafe](http://openjdk.java.net/jeps/193).

What Jigsaw has achieved is modularisation of the JDK, including the standard libraries. With the `jdeps` tool that was released in Java 8, you can quite easily work out the minimal set of platform modules needed by your application, and then ship a cut-down copy of Java along with it. This will be invaluable for desktop and mobile apps, and for Java's viability in the burgeoning IoT market.

What Jigsaw has not yet achieved -- and frankly seems to be as far off as ever -- is delivering a practical module system for applications. There are huge question marks over whether the uncompromising inflexibility of Jigsaw can ever work for Java EE, or Spring, or CDI, or Hibernate, or dynamic languages like Groovy and JavaScript. This much is apparent to anybody browsing the mailing lists. Unfortunately Oracle is holding up the release of the modularised JDK while they try to figure out all of the application-level problems.

Worse, Jigsaw is already quite complex and will only grow in complexity as it is forced to support code patterns that are widely used in the Java ecosystem. Jigsaw is repeatedly touted as a "simple" module system... simpler than OSGi, which is perceived as complex. But OSGi did not set out to be complex: it is only as complex as it needs to be in order to work in the messy real world. When Jigsaw reaches a comparable level of complexity, what will be the excuse for choosing it over OSGi?

OSGi is *clearly* a better module system for applications. It is dynamic, it has a powerful service registry, and has a dependency model that makes sense and is amenable to automated reasoning. As I proved in my [last blog post](/2015/11/13/osgi-jigsaw.html), it works on Java 9 and can take advantage of the modular JDK. And of course, it has been maturing and solving real world problems for over 15 years. However I have long acknowledged that the JDK itself has some unique problems that were difficult to solve with OSGi, which is why Jigsaw was needed. Back when Jigsaw slipped from Java 8, even Oracle's Senior VP of Development [agreed](http://www.infoq.com/news/2014/07/Project-Jigsaw-On-Track-Java-9#anch112436) that this combination was a good solution.

I think that the Java community would be better served by releasing Java 9 on time, including the modularised JDK provided by Jigsaw. Java today competes against many languages and platforms that are easier to use (or at least easier to *start* using), cooler, smaller, and fast enough. It can stay relevant by slimming down, speeding up and implementing more useful features -- not reimplementing poor replicas of features already developed years ago by the community.

Jigsaw is great. It’s time to declare victory, release it, and move on.

