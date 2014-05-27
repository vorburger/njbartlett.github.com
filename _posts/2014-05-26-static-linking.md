---
layout: post
title: Static Linking in OSGi
summary: When, why and how to do it
tags: osgi bnd
date: 2014-05-26 12:00:00
---

When building native executables in a language like C or C++, we have a choice about how to deal with the libraries our code depends on. We can choose to link them statically, meaning that the library contents are physically copied into the executable file at build time, or we can link dynamically which means the library contents must be found externally to the executable by a special runtime component called the linker.

Static linking has the great advantage that the executable can rely on always having the library it needs, and it’s always the correct version. Installation is a breeze because there’s just a single file to copy. The clear downside of static linking is that commonly used libraries will be duplicated in lots of executables, and therefore take more disk space and perhaps memory.

In standard Java, a JAR is a kind of executable that has dependencies on libraries. However static linking is unheard of in this world because it is terribly unsafe. All JARs sit together in a global classpath -- if one JAR contains a copy of a library and another JAR contains a copy of the same library then the first one on the classpath will always “win”. If the two copies are of different versions then chaos ensues.

Therefore the state of the art in standard Java is a kind of dynamic linking, except without any metadata in the JARs to declare exactly what we should be linked with… and no linker! Instead, developers often have find dependencies through a process of trial-and-error: keep adding JARs to the classpath until all the `NoClassDefFoundErrors` go away.

OSGi applications predominantly use dynamic linking as well. It’s a lot more manageable because we have precise metadata defining the dependencies, and the OSGi Framework itself can be seen as a kind of linker. However, static linking is also perfectly possible and safe because of the isolation provided by OSGi, and it has many of the same advantages as statically linked native executables. Bundles with fewer dependencies are simply easier to manage. To put it another way: OSGi is the best system I have ever seen for managing dependencies, but the easiest artefacts to manage are still those with no dependencies at all.

Static linking in OSGi is sometimes done by means of the `Bundle-ClassPath`, which allows us to embed an entire library JAR inside a bundle. This works, but it can inflate the size of our bundle quickly because we pull in the entire JAR rather than just the parts we need. An alternative solution is to use `Private-Package` to include individual packages out of the dependency JARs. However once a package is added this way it becomes a permanent part of our bundle until we explicitly remove it from the `Private-Package` list. In the future our core code might change such that some or all of these packages will not be needed. Will we remember to review `Private-Package` regularly to ensure everything there is actually still used? Not likely.


### A Better Way

And so we come to a little-known bnd instruction called `Conditional-Package`. This instruction is used to include a package or set of packages if (and *only* if) they are used from the core packages of our bundle -– where core packages are defined as those that are listed in `Private-Package` and/or `Export-Package`. We can even use wildcards, and bnd will calculate the minimum set of packages that need to be included. For example, many of my bnd files include a line like this:

{% highlight text %}
Conditional-Package: org.example.util.*
{% endhighlight %}

where the `org.example.util` prefix is for a whole set of packages designed as generic utilities.

### Service vs Library


Static linking with `Conditional-Package` can help to resolve a common head-scratcher when designing OSGi bundles.

Sometimes we want to write a piece of utility code that will be used across a number of other bundles, but that doesn't seem to warrant a service interface. Services are great for application components that can be implemented in more than one way, but sometimes there is only one conceivable implementation. A classic example is encoding, e.g. Base64. When a client wants to convert to/from Base64 representation, it doesn't want to have an alternative encoding swapped in at runtime! In this case the client is quite happy to be tightly coupled to a specific implementation of the function it is calling.

It feels wrong to export a package that contains implementation code in OSGi. Ideally, exports are for service interfaces, i.e. *contracts*. Implementation code is subject to rapid change and is hard to version correctly (tools like bnd's baselining can't detect semantic changes in executable code, only on method signatures). Nevertheless we still want to reuse the Base64 encoder across all the bundles that need it. So we use `Conditional-Package` to *include* that functionality directly. No versioning headaches because the bundle always gets the exact version was built with.

Incidentally this practice doesn't violate the DRY (Don't Repeat Yourself) principle because the source code is still defined in just one place; it's the compiled class files that are copied.

### Contraindications


A few words of warning. Like all power tools, `Conditional-Package` is dangerous if misused. When we pull a package into our bundle we inherit all of its dependencies, so it is best to do this with small, coherent packages that do not have a large tree of transitive dependencies. Try to design your packages so that they do only one thing each... don't make a big generic `util` package, instead make subpackages such as `util.xml`, `util.io`, `util.encoding` and so on. Note that `Conditional-Package` will pull in any transitive package dependencies that match the pattern we have specified -- all others are treated as external dependencies and end up in `Import-Package`.

Using a qualified or prefixed wildcard such as `org.example.util.*` is okay. Using a bare wildcard is **not** okay. I have seen this done accidentally: as a result, the bundle contained large chunks of the JRE as well as OSGi core packages.

Another danger is if we pull in packages that are used by an exported interface. Take the following example:

{% highlight java %}
package b;
import a.A;
	
public interface B {
     void doSomeThing(A a);
}
{% endhighlight %}

This should be familiar to OSGi developers as a _uses_ constraint, because `B` communicates type `A` from package `a`. It would be very bad to include a private copy of package `a` in our bundle, because it would not be compatible with other bundles' idea of what the package looks like. Actually bnd will report a build warning when it sees an exported package having a _uses_ constraint on a private package, so you easily can avoid this situation.

### Conclusion

Static linking in OSGi can improve your bundles, making them safer and easier to deploy. As always there is a trade-off and the technique can lead to larger bundles overall, but this can be mitigated by keeping packages small and coherent.

