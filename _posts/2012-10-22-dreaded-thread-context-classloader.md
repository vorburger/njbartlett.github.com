---
layout: post
title: The Dreaded Thread Context Class Loader
summary: A modular perspective
tags: java osgi eclipsecon
date: 2012-10-23 12:00:00
---

Tomorrow at <a href="http://www.eclipsecon.org/europe2012/">EclipseCon Europe 2012</a> I will be talking about <a href="http://www.eclipsecon.org/europe2012/sessions/how-make-your-code-osgi-friendly-without-depending-osgi">"How to make your library OSGi friendly without depending on OSGi"</a>. During this talk I will make reference to Java's Thread Context Class Loader (TCCL), but since it is only a 25-minute talk I will not have time to go into great detail on this subject. Therefore I thought it would be helpful to write this post with some of the background information.

Much of the technical information and research in this post is derived from an old, unpublished OSGi RFP authored by Peter Kriens. I have used it with his permission.

## History

The thread context class loader (TCCL) has a very interesting history in Java.

Java defines a hierarchy of class loaders, and in a typical Java runtime the "application" loader at the bottom is aware of the "classpath" (as supplied via the `-classpath/-cp` switch), and the boot loader is aware of the JRE classes in rt.jar etc. There is also an "extension" class loader in the middle, which loads from the JRE `ext` directory. Class loaders delegate upwards to their parent, which means the application loader can see any class offered by the boot loader but the boot loader cannot see the application classes.

Sun first ran into problems with this scheme when implementing Java Serialization, as used in Remote Method Invocation (RMI). Deserializing data from a stream as Java objects required knowledge of the application classes, but the deserialization code was loaded by the boot loader; therefore it would not normally have visibility of the classes. The problem was fixed by adding private native methods to the JRE that would allow inspection of the call stack in order to find the first non-null class loader.

However, many other extension libraries soon ran into the same problem, and they could not (and should not!) all solve it with private native methods in the Sun JVM. At about the same time, J2EE became very important. J2EE is an application model where the Java code runs in a severely restricted environment. Applications run in a "silo". Each application runs in a separate class loader and threads may not cross application/silo boundaries. Applications can use the extensions provided by the VM and the libraries provided by the container, but they cannot use any code from each other. This model also suffers from the problem that the libraries provided by the container cannot access the application code.

Therefore in Java 1.2 the TCCL was introduced: a class loader that is associated with a thread, as a thread-local variable. This means that any libary can at any time access the "current" context loader by asking the calling thread. This context loader is expected to have visibility to the specific application class loader that is responsible for the call, and therefore visibility of the application classes.

Sun also started to use the TCCL heavily itself, though without proper specification or guidance. The 1.5 JDK has 79 references to `Thread.getContextClassLoader()`. Since Java 1.4, the Java VM was modified to use the context class loader in JNDI, JAXP, CORBA, JMX, Xalan, Xerces, AWT, Beans, SQL, Logging, Prefs, RMI, Security, Swing, and most of the XML subsystems. Additionally most middleware libraries like Hibernate, Saxon, Jakarta Commons Logging and many others started to use the TCCL as either the first or last resort. Java has provided virtually no specification or guidelines on the use of TCCL. This has caused almost every library to follow a different strategy.

The key questions to answer regarding the correct use of TCCL are:

* when should it be set, and by whom?
* of what set of classes should it have visibility?

From a J2EE point of view these questions are relatively easy to answer because the programming model is so constrained. The container owns all the entry points; i.e., it controls the HTTP thread running the Servlets and the RMI sockets for EJBs, creation of additional threads is forbidden, etc) and it can therefore ensure that the TCCL is always set on entry into the application code. Since applications are entirely self-contained, the set of visible classes is simply all classes owned by the application.

In a modular runtime such as OSGi these questions are far harder to answer. Bundles are free to create their own threads and entry points, and there is no general way to identify the set of bundles that contribute classes for an "application"; indeed, the very concept of an "application" is much harder to pin down. We could constrain the programming model and require that an application be deployed as a single bundle with no dependencies, but this would negate most of the benefits of using OSGi. Therefore OSGi does not attempt to define what the TCCL should be at any particular time, and typically it is left null.

## Alternatives to TCCL

In our own OSGi-compliant code, or in any library with explicit OSGi support, we need not worry about the TCCL, because OSGi Services provide a far cleaner approach than any ClassLoader-based approach to loading extensions. However, legacy third-party libraries are still an issue. Many such libraries attempt to load application classes by name at runtime: for example, Hibernate reads class names from `.hbm.xml` files and then creates instances of those classes for each database record.

When you move such a library to *any* kind of modular environment -- including OSGi, JBoss Modules or even Jigsaw -- you find that it breaks, because the name of a class is not sufficient to uniquely identify it. The identity of a class consists of its fully qualified name AND the class loader that defined it, which in OSGi usually corresponds to the bundle that contains it.
So in addition to the name, we need to know the class loader from which to load the class. Due to the wide variety of class loading environments created by various application servers, many libraries attempt to solve this problem with a set of heuristics. Consulting the TCCL is usually one of the heuristics, along with checking the library's own class loader, the JRE extension class loader, etc.

If a library *only* consults the TCCL then we are somewhat stuck: we will have to explicitly set the TCCL from our own code before calling into the library. Fortunately this is rarely the case, for example most such libraries will also call `Class.forName()`, which means the library will consult its own class loader to load the class. This is still far from ideal, but we can work around it by deploying a single fragment rather than scattering calls to `setContextClassLoader()` throughout our code. Far better is when a library eschews class names entirely and allows Class *objects* to be passed; or at least offers an API method to set the class loader from which the class names should be loaded.

Unfortunately it's very hard to predict in advance -- at least without closely inspecting the code -- what kind of heuristics the library employs, again because of the complete lack of specification and guidance in this area.

## Summary

I hope you will attend my talk tomorrow, which is mainly addressed to authors of Java libraries that want their code to work well in OSGi, but not *only* in OSGi. As you can doubtless infer from this blog post, avoiding the TCCL and other weird ClassLoader-based hacks is a big part of being OSGi-friendly, but I will also talk about service dynamics and configuration issues. See you there!