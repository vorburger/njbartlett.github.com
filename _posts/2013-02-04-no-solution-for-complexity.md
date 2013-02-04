---
layout: post
title: No Solution for Complexity?
tags: java osgi modularity
date: 2013-02-04 12:00:00
---


Yesterday I read a fascinating [article on the BBC News website](http://www.bbc.co.uk/news/technology-21280943) about the complexity of banking software. To summarise: last year saw a number of severe IT glitches at banks causing outages in payment processing and other problems, and the article predicts that such glitches will get more common in the future due to increasing sofware complexity.

Here's the part that got my attention:

>The core of the problem is that the business software used by the institutions has become horrifically complex, according to Lev Lesokhin, strategy chief at New York-based software analysis firm Cast.
>
>He says developers are good at building new functions, but bad at ensuring nothing goes wrong when the new software is added to the existing mix.
>
>"Modern computer systems are so complicated you would need to perform more tests than there are stars in the sky to be 100% sure there were no problems in the system," he explains.

Sigh.

If only there were a way to create "firewalls" between different parts of a large system, so that we could be absolutely sure that the functionality within each firewall cannot break merely from adding new functionality outside it. Then we could know precisely the scope of any change, and test only the things that can potentially break rather than the entire universe.

Of course *there is* a way to do this. It's called **modularity**. Sadly it's not very popular because it requires planning and discipline -- two words that developers seem to be averse to, even though they pay off handsomely by averting cock-ups in deployment. Also to be fair to the developers, it requires strategic investment in infrastructure, which has been sorely lacking. Businesses always have difficulty justifying measurable up-front costs leading to unknown future gains. So let me tell you a story about real, measurable gains from modularity.

A few years ago I worked for a manufacturer of medical devices. These were large, powerful machines that could quite easily kill a patient if they were to malfunction. Incidentally, [Mike Milinkovich](https://mmilinkov.wordpress.com/) tells a terrifying story about how he was nearly killed by a [software bug](https://en.wikipedia.org/wiki/Therac-25)... all things considered I prefer banks; when they go wrong the only thing you lose is money.

Naturally, the business was heavily regulated. Whenever a module of the system (hardware or software) was changed, it would have to be thoroughly retested, and those tests certified by the regulator. The process was terribly slow and expensive, and it would have to be done not just for the changed module but for all the other modules that could potentially be affected by it. By using a strong modularity solution, the manufacturer could *prove* to the regulator that a change in module A could not possibly affect the working of module B, and therefore module B did not need to be retested. The engineering effort required to achieve this was not cheap, but it ultimately saved them millions of dollars and allowed them to bring new features to market quicker.

Modularity is a technique and a mindset as much as it is a specific technology. Nevertheless there is one technology that is mature, well proven and has stood the test of time: [OSGi](http://www.osgi.org/Main/HomePage). Another quote, this time from [Peter Kriens](http://softwaresimplexity.blogspot.co.uk/):

>OSGi ... is a messenger that tells you that your code is ugly. In more traditional environments you would have this fuzzy warm feeling that all was ok due to the absence of warnings and errors. The power of OSGi is that it actually allows you to go to a warm and fuzzy place that is not fiction (like all other Java build environments) but based on hard facts.

Look it up, and perhaps you could save your bank from making headlines for all the wrong reasons.