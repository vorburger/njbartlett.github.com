---
layout: post
title: What's New in Declarative Services 1.3?
summary: Cool new stuff in OSGi Release 6
tags: osgi bnd
date: 2015-08-17 12:00:00
---

**I'd like to tell you about two really cool new features in OSGi Declarative
Services**. They're so cool that I feel kind of guilty for blogging about them,
because some of the kudos may accrue to me even though I have done absolutely
none of the work to either specify or implement them. So bear in mind that I'm
only your humble messenger.

### Configuration Property Types ###

As a bit of background, in Declarative Services (DS) it has always been possible to write configurable
components by passing a Map into the activate method:

{% highlight java %}
@Component
public class ServerComponent {

  @Activate
  public void activate(Map<String, Object> configProps) {
    String bindHost = (String) configProps.get("host");
    int port = (Integer) configProps.get("port");
    
    ServerSocket sock = new ServerSocket();
    sock.bind(new InetSocketAddress(bindHost, port))
    // ...
  }
}
{% endhighlight %}

There are some great things about the way DS handles configuration. First, the
component itself has no idea about the origin of the config data... it could
come from a properties file, or an XML file, or a database record, or
transmitted over the air from a cellular provider, etc. Because the component
is not coupled to the physical mechanism, we can change that mechanism easily
without updating the code of all our components.

Second, the configuration is passed all-at-once and can be dynamically changed.
Some other DI frameworks inject configuration data one field at a time
(`setHost`, `setPort`...). Where they support dynamic reconfiguration at all, they
tend to need methods to let the component know that a series of config changes
is starting and ending, so that it knows when it's safe to reconfigure. Otherwise
the component could end up using a mixture of old and new config.

There is a problem, however. Did you notice that my code sample above had no
error handling at all? What if the `host` property was missing from the map?
What if the port was given as a String rather than an integer?  What if it
contained invalid numeric characters? Some fields may be optional, where do we
specify the defaults? It turns out that most components have to do quite a bit
of work to convert, parse and validate their configuration. There are libraries
that encapsulate this -- such as the Configurable API from bnd -- but
they add a runtime dependency, which is inconvenient (though manageable in OSGi
with [static linking](/2014/05/26/static-linking.html)).

Declarative Services 1.3 -- part of the OSGi Release 6 compendium spec, which
was released to the public last week -- improves this enormously. It's now
possible to define a custom type for the configuration, rather than a generic
Map. For example:

{% highlight java %}
@interface ServerConfig {
  String host() default "0.0.0.0";
  int port() default 8080;
  boolean enableSSL() default false;
}

@Component
public class ServerComponent {

  @Activate
  public void activate(ServerConfig cfg) {
    ServerSocket sock = new ServerSocket();
    sock.bind(new InetSocketAddress(cfg.host(), cfg.port()));
    // ...
  }

}
{% endhighlight %}

Now the DS runtime is doing all the work of pulling out fields, converting them
to the correct type and so on. If there are any conversion errors, or missing
fields that don't have defaults, then DS will refuse to load our component and
will write an appropriate message into the OSGi Log.

This feature is known as "Configuration Property Types". You may wonder why
an annotation is used rather than an interface, given that we don't actually
use it like an annotation. There are a few reasons: in
annotations it's trivial to provide defaults; the methods cannot have
parameters; and the return types allowed in an annotation are restricted to
types that mesh well with configuration data.

Of course this version of DS interoperates with the OSGi Metatype specifcation,
so that the administrators of our application can know what fields and types are
actually expected. The metadata is generated at build time if we just add a couple
more annotations:


{% highlight java %}
@ObjectClassDefinition(name = "Server Configuration")
@interface ServerConfig {
  String host() default "0.0.0.0";
  int port() default 8080;
  boolean enableSSL() default false;
}

@Component
@Designate(ocd = ServerConfig.class)
public class ServerComponent {
  // ...
}
{% endhighlight %}

... and, from this definition, tools like the Apache Felix Web Console can automatically
generate an admin GUI for our component:

<img src="/images/posts/ds1.3/webconsole.png" width="499" height="185">

### Prototype Scope Services ###

The second really cool feature is support for *prototype scope* in services.
The prototype scope has been available in the OSGi Core R6 specification for
about a year now, but there was a significant gap between the release of the
Core and Compendium documents. So until now we had to drop down to the
low-level OSGi API in order to use prototype scope. In DS 1.3 we can use
annotations and a more convenient runtime API instead.

Prototype scope is where a service has the ability to create multiple instances
on demand. Traditionally, most OSGi services were conceptually singletons: each
registry entry corresponded to a single back-end service instance that was
shared by all consumers. (Aside: there were additionally bundle-scoped
services, where each consumer bundle got its own instance of the service
object, but if you used a service multiple times from within the same bundle
you would still share the same instance).

This model had to be enhanced, since sometimes each consumer really needs to
have its own instance of the service. Also some consumers need to have
programmatic control over the creation and destruction of instances, in order
to fit in with an external lifecycle. An example of this would be in web
requests -- some web standards (for example JAX-RS) require services to be
instantiated and then destroyed in every request. Hence the prototype scope was
added. This is an opt-in feature: both the provider and the consumer of the
service need to be aware that it is being used.

In DS 1.3, a provider can opt in to the prototype scope very easily with an
attribute on the @Component annotation:

{% highlight java %}
@Component(scope = ServiceScope.PROTOTYPE)
public class MyComponent {
  // ...
}
{% endhighlight %}

As a consumer there are a few more choices. When we consume a service with the
@Reference annotation, we can use an attribute to request a new instance per
component:

{% highlight java %}

@Reference(scope = ReferenceScope.PROTOTYPE_REQUIRED)
public void setFoo(Foo foo) {
  // ...
}

{% endhighlight %}

Using this annotation, each component that uses the Foo service will get its
own instance. However within each component, the same Foo instance will be used
repeatedly. Alternatively, the consumer can create and destroy instances
programmatically whenever it wants -- this requires the consumer to use a bit
of API from DS as follows:

{% highlight java %}

private ComponentServiceObjects<Foo> fooFactory;

@Reference(scope = ReferenceScope.PROTOTYPE_REQUIRED)
public void setFooFactory(ComponentServiceObjects<Foo> fooFactory) {
  this.fooFactory = fooFactory;
}

public void doSomething() {
  Foo myFoo = fooFactory.getService();
  try {

    // do something interesting with myFoo ...

  } finally {
    fooFactory.ungetService(myFoo);
  }
}


{% endhighlight %}

The use of the ComponentServiceObjects interface means we are no longer
building strict POJOs, but still this interface is easily mockable for testing
purposes.

### Getting It ###

The specification for DS 1.3 is available now from [http://www.osgi.org/Download/Release6](http://www.osgi.org/Download/Release6). Felix SCR 2.0 implements the specification and can be downloaded from [http://felix.apache.org/downloads.cgi](http://felix.apache.org/downloads.cgi). For full metatype support you will need Felix Metatype 1.1 from the same page.

To build components using the annotations you will need a development build of bnd/Bndtools 3.0. To install this, go to [http://bndtools.org/installation.html](http://bndtools.org/installation.html) and follow the instructions for installing a developer snapshot build.
