---
layout: post
title: Vertical Components
summary: Combining Web and OSGi Components
tags: osgi component web
date: 2016-05-06 10:00:00
comments: true
---

### Combining Web and OSGi Components

I consider myself a latecomer to Microservices; I only really started to understand and use them in 2005. Nobody called them "Microservices" at the time -- as far as I can tell, the term was coined in 2006 by BEA for their [microService Architecture (mSA)](https://web.archive.org/web/20061017101304/http://www.bea.com/framework.jsp?CNT=msa.jsp&FP=/content). Regardless, with this history it was a little surprising to hear the term reinvented in 2014 and suddenly gain a great deal of traction. While implementation details vary, the fundamental concept has not changed: small, independently deployable, discoverable services.

I'm always keen to hear from people with a different take, and so in 2014 I found myself listening to a [Software Engineering Radio podcast interview with Stefan Tilkov](http://www.se-radio.net/2014/09/episode-210-stefan-tilkov-on-architecture-and-micro-services/), an expert on REST and software architecture, on the subject of microservices. One of his ideas as elaborated on the podcast was to use microservices to slice applications vertically, rather than the horizontal layering that is much more commonly seen in enterprise architectures. He promotes the idea of a _Self-Contained System_ that includes all of the logic, data and UI for a single business domain. Inclusion of the UI is what makes the approach unconventional: componentisation is relatively commonplace in the back-end, but does not often extend all the way up to the user. This is a fine idea, as it allows technical teams to align with business units. When building horizontal layers (data access layers, application servers, ESBs, etc) developers easily get sidetracked into endlessly polishing frameworks rather than delivering real business value.

The problem was, at least as I saw things at the time, that the UI tier was hard to componentise in this way. The web is by far the dominant UI platform, but web modularity has historically been difficult. The traditional unit of deployment and isolation in the web is the HTML page, but in a modern web application the page is the whole application. Frameworks like Angular and React help in wiring together single-page applications, but didn't really allow applications to be composed out of independently deployable components. Also, crucially, code written for Angular is useless in a React app and _vice versa_.

Of course, I was missing the [Web Components](http://webcomponents.org) family of specifications. In my defence, I don't think many people knew about Web Components, as all the noise seemed to be around the competing frameworks. Also they were not really practical until recently due to the lack of support in browsers. As I write this in mid-2016, however, things have moved forward and Web Components seem to finally be ready for production. This means we can now write independently deployable and reusable components that can be assembled into an application. There are various libraries for building components, but the specifications allow (theoretically!) for components to interoperate irrespective of their implementation details. The application developer can still use a framework to assemble these components into a complete application.

The parallels with OSGi components are very clear. So I started thinking about whether we can combine the two worlds to define _vertical components_. A web component is often required to invoke logic or retrieve data from a back-end server, which creates a coupling. So what if we could create an artifact that deploys both the front-end and the back-end components together? And then what if we could use the OSGi Resolver to assemble an application out of its required UI and back-end components? So that's what I have experimented with, and the results seem quite encouraging.

I have shared a [proof-of-concept on GitHub](https://github.com/njbartlett/demo-vertical-components), which implements a very simple Shopping Cart component that displays the content of the customer's cart, retrieved from a back-end REST service.

### Web Component Code

I have tried to use as few libraries and assistive technologies as possible, in order to show that nothing magical is happening. So, while libraries like [Polymer](https://www.polymer-project.org/1.0/) or [Bosonic](https://bosonic.github.io) are handy for creating web components, it's also pretty easy to define a simple component in plain vanilla ES2015:

{% highlight javascript %}
(function() {
  let template = `
    <div class="cart-list">
      <h2>Cart Contents:</h2>
      <table class="items-table"></table>
    </div>
  `;
  
  class CartListWidget extends HTMLElement {
    createdCallback() {
      this.createShadowRoot().innerHTML = template;
      this.$table = this.shadowRoot.querySelector('.items-table');
      this.requestServer();
    };
    requestServer() {
      var that = this;
      var xhr = new XMLHttpRequest();

      xhr.open('GET', '/cart-list/servlet/');
      xhr.onload = function() {
        var cart = JSON.parse(xhr.responseText);
        var rows = '';
        for (var i = 0; i < cart.length; i++) {
          var row = '<tr><td>' + cart[i].sku + '</td><td>'
                  + cart[i].price + '</td></tr>';
          rows += row;
        }
        that.$table.innerHTML = rows;
      };
      xhr.send();
    };
  }
  document.registerElement('cart-list', CartListWidget);
})();
{% endhighlight %}

### OSGi Component Code

The JavaScript file is served from an OSGi bundle that also provides the REST service. I could have used JAX-RS for this, but a plain old Servlet works just as well for such a small demo. I even build the JSON response with printf!

{% highlight java %}
@Component(
    property = {
      HTTP_WHITEBOARD_SERVLET_PATTERN + "=/cart-list/servlet/*",
      HTTP_WHITEBOARD_RESOURCE_PATTERN + "=/cart-list/*",
      HTTP_WHITEBOARD_RESOURCE_PREFIX + "=/cart-list"
    })
@WebComponent(name = "cart-list")
@RequireHttpImplementation
public class CartServlet extends HttpServlet implements Servlet {

  private static final long serialVersionUID = 1L;
  
  @Reference
  private CartManager manager;

  public void doGet(HttpServletRequest request, HttpServletResponse response)
                                        throws ServletException, IOException {
    try (PrintStream out = new PrintStream(response.getOutputStream())) {
      Cart cart = manager.getCart("anonymous");

      StringBuilder builder = new StringBuilder();
      builder.append("[");
      int row = 0;
      for (CartEntry entry : cart.listEntries()) {
        if (row++ > 0) builder.append(',');
        builder.append(String.format("{\"sku\" : \"%s\", \"price\" : %d}",
                       entry.getSku(), entry.getSoldPrice()));
      }
      builder.append("]");
      response.setHeader("Content-Type", "application/json");
      out.print(builder.toString());
      out.flush();
    } catch (Exception e) {
      throw new ServletException(e);
    }
  }
}
{% endhighlight %}

### Application Code

Finally, the application bundle contains `index.html` which simply uses the `cart-list` component we defined:

{% highlight html %}
<!DOCTYPE html>
<html>
<head lang="en">
	<meta charset="UTF-8">
	<title>Bookstore App</title>
	<script src="webcomponents.min.js"></script>
	<script src="/cart-list/component.js"></script>
</head>
<body>
	<h1>Book Store</h1>
	<cart-list></cart-list>
</body>
</html>
{% endhighlight %}

<p></p>

### Resolution

In the code there are a couple of new annotations, which I have adapted from <a href="http://enroute.osgi.org">OSGi en Route</a>. In the web component bundle, the `@WebComponent` annotation generates a `Provide-Capability` header in the OSGi bundle manifest that declares our bundle provides the `cart-list` web component. The `@RequireHttpImplementation` generates a requirement for an HTTP Service implementation.

In the application bundle we use `@RequireWebComponent` to require the component that is provided by the web-component bundle. The application bundle also repeats the `@RequireHttpImplementation` annotation for completeness.

These annotations are used by an OSGi Resolver, such as the build-time resolver in <a href="http://bndtools.org/">Bndtools</a> or the runtime resolver in <a href="https://paremus.com/products/">Paremus Service Fabric</a>. Because of this, we don't need to manually list all of the components and libraries used by the application... we only need to list the top-level bundle (in this case `org.example.webapp.bookstore` which contains the `index.html`) and the tooling will automatically pull in the `cart-list` component bundle along with an HTTP Service implementation such as Jetty.

### Conclusion

This is just a proof-of-concept and there are many details that could be improved. For example, the mapping between the component name and the URL of the JS and Servlet could be handled by a small server-side framework. Using third-party components from Bower or NPM could be made simpler, perhaps via an extender bundle.

What do you think... is this a useful approach? What flaws have I missed? Let me know in the comments, or using the contact details to the right.

If you are interested in learning more about web development with OSGi, and about modern component-oriented development with Java, then please consider supporting my new book, [Effective OSGi](https://www.indiegogo.com/projects/effective-osgi), or buying a copy when it is published!