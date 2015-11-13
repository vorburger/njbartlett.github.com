---
layout: post
title: OSGi and Java 9 Modules Working Together
summary: A demonstration of OSGi/Jigsaw interoperability
tags: osgi jigsaw bnd
date: 2015-11-13 12:00:00
---

The best thing about the [Java Platform Module System](http://openjdk.java.net/projects/jigsaw/spec/sotms/) -- commonly known as Project Jigsaw, or [JSR 376](https://www.jcp.org/en/jsr/detail?id=376) -- is that it has been used to break apart the monolithic Java runtime into smaller modules. This makes Java a more attractive implementation technology in many scenarios where it has historically been weak. For example, modern desktop or mobile applications can't rely on having a pre-installed copy of Java on the user's device, so you need to ship Java as part of your application. IoT is also driving developers to seek smaller installed size and memory footprint.

Unfortunately, Jigsaw has some shortcomings as a module system for applications. I don't want this blog post to be a critique of Jigsaw, but suffice to say there are many developers who will prefer to use OSGi for application modularity.

Still, it would be very useful if we could run a modular OSGi application on a modular Java runtime. This would allow us to benefit from a minimal Java install, and at the same time use the full power of OSGi. Also it would help to hide much of Jigsaw's complexity from application developers.

I have just spent a day hacking together a [demo](https://github.com/njbartlett/osgi_jigsaw) that proves you can do this.

I must emphasise that this is a throw-away prototype. You should **absolutely not** start building on it. The OSGi Core Platform Expert Group (CPEG) will be working to specify official support for Java 9 features... I just wanted to show that a solution can exist, and perhaps feed into their discussions.

NB: The following description may get confusing because of the overlapping concepts used by Jigsaw and OSGi, so I have to be very careful with terms. Whenever I say "module" I always mean a Jigsaw/Java 9 module, and when I say "bundle" I'm always talking about OSGi.

## Use Case: OSGi Bundles in a Known Platform Configuration  ###

Suppose we have a Java runtime containing a known set of modules. We would like to start an OSGi Framework within the context of a module, such that the bundles in that framework can import packages from the platform modules.

As background, OSGi has long used the concept of an Execution Environment (EE), which models the Java platform on which the framework is running. These have names such as `JavaSE-1.7`. When an OSGi framework starts up, it detects what kind of platform it is running on and declares the EE as a capability that bundles can depend on.

If you are unfamiliar with the term "capability" in this context, it is basically how all dependencies work in OSGi. Any bundle can declare a capability, which has a namespace and a set of attributes. Then any other bundle can require a capability, optionally with a filter expression. The OSGi Framework ensures that for every requirement, there exists a bundle providing a matching capability, otherwise the requirement will not resolve. OSGi additionally defines the meaning of some special capability namespaces. One of these is `osgi.ee`, which is declared by the OSGi system bundle to indicate the Java platform version.

Why can't we work out a bundle's platform dependency based on its imported packages? That's not possible for two reasons. First, the packages from the Java platform are not versioned. Second, a large subset of packages -- namely those starting with `java.` (e.g. `java.net`, `java.lang` etc) cannot be imported; they have to be loaded from the boot class loader. So bundles have to declare a requirement on the EE in order to ensure they can safely call methods like `Files.readAllBytes` (added in Java 7) or `String.isEmpty` (added in Java 6).

An EE is basically a monolith: Java 7 is Java 7 and there are no other flavours. Things got a bit more complicated in Java 8 with the introduction of Compact Profiles... but these are basically still monolithic platforms. With just three profiles defined in the Java 8 specification, it was no problem for OSGi to define an EE for each profile. However in Java 9 there is a very large set of potential module combinations -- too many to model with EEs.

In my prototype I posit a capability namespace of `jmodule` to describe the Java modules available in the platform. At runtime, the launcher inspects the set of available modules and uses them to declare the `jmodule` capability. The relevant code looks like this:

{% highlight java %}
// Calculate the system capabilities
List<String> systemCapabilites = new ArrayList<>();
for (Module module : Main.class.getModule().getLayer().modules()) {
    String cap = String.format("jmodule;jmodule=%s;version:Version=1.9", module.getName());
    systemCapabilites.add(cap);
}

// Configure the framework
Map<String,String> configMap = new HashMap<>();
configMap.put(Constants.FRAMEWORK_SYSTEMCAPABILITIES_EXTRA,
              formatAsList(systemCapabilites));
Framework framework = frameworkFactory.newFramework(configMap);
{% endhighlight %}

An OSGi bundle can now declare a requirement for a Jigsaw module as follows:

{% highlight text %}
Require-Capability: jmodule; filter:="(jmodule=java.xml)"
{% endhighlight %}

... and the OSGi Framework will refuse to resolve the above module if the required module (in this case, `java.xml`) is not present in the platform. This is valuable because it protects against NoClassDefFoundErrors and ClassNotFoundExceptions from occurring at some unknown time during the life of the bundle. OSGi provides the assurance that the bundle will actually work on the platform that it finds itself on.

But how do we know that our bundle depends on the `java.xml` module? At the moment we have to use the `jdeps` tool from the JDK:

{% highlight text %}
$ jdeps -module org.example.xml.jar
org.example.xml.jar -> java.base
org.example.xml.jar -> java.xml
org.example.xml.jar -> not found
   org.example.xml (org.example.xml.jar)
      -> java.io                                            java.base
      -> java.lang                                          java.base
      -> javax.xml.parsers                                  java.xml
      ...
{% endhighlight %}

... which tells us that our bundle depends on `java.base` and `java.xml`. The `java.base` module is implicit so we can ignore it, but we do need to add `java.xml` to the requirements.

This process is manual and cumbersome right now, but in the future bnd should be able to generate the requirement automatically.

We're not done yet though, because the Java platform provides a whole bunch of packages that are not in the `java.*` namespace. For example: `javax.xml`, `javax.sql.rowset`, `org.w3c.dom`, `org.omg.CORBA` etc. We don't need to do anything different in our OSGi bundles for these, because bnd will generate imports as normal. However we need the OSGi launcher to work out which packages are available from the platform modules. It can do this again by iterating over the modules using reflection:

{% highlight java %}
// NB this sample ignores package versions for now
private static List<String> calculateExports(Module module) {
    String[] packageNames = module.getPackages();
    List<String> exports = new ArrayList<>(packageNames.length);
    for (String packageName : packageNames) {
        // ignore java.* packages
        if (packageName.startsWith("java."))
            continue;

        if (module.isExported(packageName))
            exports.add(packageName);
    }
    return exports;
}
{% endhighlight %}

This is actually a massive boon to the OSGi Framework implementers. In previous Java releases, the Framework had to maintain a list of all the known packages provided by the platform; there was no standard API for enumerating classes or packages on the classpath (even if there were, it would reveal non-standard packages such as `sun.misc`, `com.sun.*` and so on). Take a look at the [properties file that Felix maintains](https://github.com/apache/felix/blob/trunk/framework/src/main/resources/default.properties) for this purpose... it's larger than you might expect.

## Implementation Issues ##

In the demo I have copied the source code for Apache Felix into my module. I did this to give myself freedom to hack the implementation, but so far have hardly needed to --- more on that shortly.

Most of the code is in [`nbartlett.osgi_jigsaw.Main`](https://github.com/njbartlett/osgi_jigsaw/blob/master/nbartlett-jigsaw-osgi/src/nbartlett.jigsaw_osgi/nbartlett/osgi_jigsaw/Main.java), and it's mostly just a vanilla OSGi launcher. The main differences are the `jmodule` capabilities and system bundle exports as already described.  The `module-info.java` looks like this:

{% highlight java %}
module nbartlett.jigsaw_osgi {	
	exports org.osgi.framework;
	exports org.osgi.framework.dto;
	exports org.osgi.framework.hooks.bundle;
	exports org.osgi.framework.hooks.resolver;
	exports org.osgi.framework.hooks.service;
	exports org.osgi.framework.hooks.weaving;
	exports org.osgi.framework.launch;
	exports org.osgi.framework.namespace;
	exports org.osgi.framework.startlevel;
	exports org.osgi.framework.startlevel.dto;
	exports org.osgi.framework.wiring;
	exports org.osgi.framework.wiring.dto;
	exports org.osgi.resource;
	exports org.osgi.resource.dto;
	exports org.osgi.service.packageadmin;
	exports org.osgi.service.startlevel;
	exports org.osgi.service.url;
	exports org.osgi.service.resolver;
	exports org.osgi.util.tracker;
	exports org.osgi.dto;
 }
{% endhighlight %}

All of the packages of the OSGi Framework have to be exported by the module wrapping it. This is necessary because the bundles actually load into a different module -- the "unnamed" module -- and so we need Jigsaw to permit access to the OSGi API packages. If we fail to do this, the following error occurs:

{% highlight text %}
java.lang.IllegalAccessError: superinterface check failed: class org.example.Activator
(in module: Unnamed Module) cannot access class org.osgi.framework.BundleActivator (in
module: nbartlett.jigsaw_osgi), org.osgi.framework is not exported to Unnamed Module
{% endhighlight %}

After fixing this, I encountered the following error when the OSGi Framework tried to instantiate the `BundleActivator` from any bundle:

{% highlight text %}
java.lang.IllegalAccessException: class org.apache.felix.framework.Felix (in module
nbartlett.jigsaw_osgi) cannot access class org.example.Activator because module
nbartlett.jigsaw_osgi does not read <unnamed module @5cee5251>
{% endhighlight %}

This one was quite a puzzle! It turns out that if you create an instance of `ClassLoader` and use it to load classes dynamically, those classes will always belong to the unnamed module. In order to load classes into a real module you have to use `ModuleClassLoader`. However this class is final, so OSGI cannot override `loadClass` to implement OSGi loading rules.

Incidentally, the upshot is that **any** code that looks like the following -- and this is a very common code pattern -- will fail from inside a module other than the unnamed module:

{% highlight java %}
URLClassLoader loader = new URLClassLoader(urls, this.getClass().getClassLoader());
Class<?> clazz = loader.loadClass("blah.Blah");
clazz.newInstance(); // <-- this line fails from within a named module
{% endhighlight %}

It fails because `blah.Blah` belongs to the unnamed module, and our module doesn't require unnamed. You can't write `requires unnamed` in the `module-info.java`, but you can dynamically add the requirement as follows:

{% highlight java %}
this.getClass().getModule().addReads(loader.getUnnamedModule());
{% endhighlight %}

For now, we have to accept that OSGi bundles will load into the unnamed module. It's an open question whether we can map them into the OSGi module, or even a module for each bundle. By adding the above line to Felix (`BundleWiringImpl` line 731), the Framework is able to load the bundle activator, and OSGi starts up:

{% highlight text %}
g! lb
START LEVEL 1
   ID|State      |Level|Name
    0|Active     |    0|System Bundle (0.0.0)
    1|Active     |    1|Apache Felix Gogo Command (0.14.0)
    2|Active     |    1|Apache Felix Gogo Runtime (0.12.1)
    3|Active     |    1|Apache Felix Gogo Shell (0.10.0)
    4|Active     |    1|org.example.xml (0.0.0.201511120014)
{% endhighlight %}

In my demo I installed and started the three bundles of Felix Gogo, which gives me an interactive shell, and a fourth bundle that contains some XML parsing code. This code forces a dependency on the `java.xml` module and the `javax.xml.parsers` package from the platform, both of which are satisfied by the system bundle.

## Use Case: Configuring the Platform for a Set of Bundles ##

Above we had a known platform and wanted OSGi to validate the bundles we wanted to run on it. Flipping this around, what if we have a set of bundle -- our application -- and want to work out the minimal platform that will run it?

You can already do this -- kind of -- with existing tools. The jdeps tool can print the module dependencies of each of your OSGi bundles, so the modules you need in your platform are just the aggregate of those jdeps outputs. The OSGi framework module I defined above doesn't have any dependencies aside from `java.base`.

If you build your bundles using the future version of bnd that generates `jmodule` requirements for you, then you don't need to run jdeps. It will be very simple to generate the aggregate dependencies just by looking at the bundle manifests. We can create a tool that builds an application artifact, including the minimal Java runtime modules plus OSGi bundles, as part of bnd/Bndtools. We already have an exporter for "standalone" JAR files, this would just be an extension of the same principle. But that's for the future.

## Open Questions ##

Here are some of the issues I haven't even touched on, that will need a good answer before OSGi and Java 9 Module interop is a reality:

1. **Versions!** Jigsaw chose to ignore versions, so they will have to be layered on top somehow. This sounds like a recipe for lots of competing and incompatible schemes... the Maven way, the Gradle way, the OSGi way, etc. Should OSGi infer versions for the exports from the platform? If so, where should those come from?
2. **Bundle-Module Mapping**. The bundle classes load into the unnamed module, meaning that we have exactly as much protection and access control between bundles as OSGi has always had -- no more, no less -- and that's fine. However, by using `ModuleClassLoader` we may be able to map each bundle to a Jigsaw module and benefit from increased isolation. I didn't have time to figure out whether this would be possible.
3. **Dynamics**. With this prototype we can install, update and uninstall OSGi bundles dynamically. But if a newly installed bundle has requirements that are not satisfied by the current set of platform modules, can we install the required module into the platform dynamically? It looks like probably not, that's just not within the scope of Jigsaw.
4. **Uses Constraints**. This is a tricky problem to describe and solve... see my [post](http://njbartlett.name/2011/09/02/uses-constraints.html) on the subject of uses constraints. Jigsaw module descriptors do not provide any information on the interdependencies between packages (e.g. the fact that `javax.sql` uses `javax.transaction.xa`, so if you import both of these packages then you really have to import them from the same source). Without this info, it will not be safe for OSGi bundles to offer exports that subtitute for exports from the platform.
5. **Services**. Jigsaw has a service model, though quite different from OSGi services. It would probably be useful for services from the platform to be bridged into the OSGi service registry. 
6. **Bidirectional Dependencies**. Can we ever build a bidirectional link between OSGi and Jigsaw, so that bundles and modules can sit side-by-side in cooperative harmony? I don't particularly care about this use case -- I believe OSGi is the better application module system so it makes sense for the dependencies to flow downwards but not upwards. However, somebody might want to do this I suppose. It will be tricky though because of the dynamics in OSGi and lack of versions in Jigsaw.
7. **Evolving EEs**. The CPEG will need to decide what to do with EEs. Maybe they will be supplemented with something like the`jmodule` capability, or perhaps something completely different. We probably still need an EE to define the version and content of `java.base`.

## Getting It ##

The prototype code is on GitHub at [njbartlett/osgi_jigsaw](https://github.com/njbartlett/osgi_jigsaw).

To build the code, execute `./build.sh` from the `nbartlett-jigsaw-osgi` directory. To run, execute `./run.sh`.

Thanks to my employer [Paremus](https://paremus.com/) for funding this research. We want to help our customers to build and deploy manageable, modular software, whatever the implementation technology.

Thanks to Sander Mak's [blog post](http://branchandbound.net/blog/java/2015/10/osgi-to-jigsaw/) for helping me to setup a Jigsaw workspace (though I can't let this pass with commenting on his last paragraph... "vastly less complex"? Seriously dude are you feeling okay?)
