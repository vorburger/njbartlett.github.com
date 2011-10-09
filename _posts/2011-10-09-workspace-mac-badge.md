---
layout: post
title: Eclipse Dock Icons on Mac OS X
summary: Sanity at last!
tags: java eclipse
date: 2011-10-09 18:00:00 +00:00
---


If you're a heavy Eclipse user on Mac OS, you're probably just as sick as I am of seeing the following in your Dock:

![Dock](/images/posts/macbadge/dock.png)

...  and the following in your Application Switcher:

![Switcher](/images/posts/macbadge/switcher.png)

How on Earth am I supposed to keep track of all those icons?? Well I **just can't**, and I've been struggling with this nonsense for years. I've tried changing the Dock icon, and the name of the application, but these settings are still shared by all the workspaces that run with that copy of Eclipse.

**Enough is enough**, I need a per-workspace label for my dock icons, so I created a plug-in that I'm calling the *Workspace Badge Plug-in for Mac OS*. It looks like this:

![Dock](/images/posts/macbadge/dock_after.png)

... and this:

![Switcher](/images/posts/macbadge/switcher_after.png)

By default it uses the last segment of your workspace path, but there isn't much space so that often gets shortened with ellipses. So you can set your own short string in the workspace using a preference page:

![Preference Page](/images/posts/macbadge/prefpage.png)

The amazing thing is, this is probably the tiniest plug-in I have ever written for Eclipse, and is likely to be one of the most useful! If you want to try it out, install from the following update site URL (Marketplace listing coming soon):

    http://macbadge-updates.s3.amazonaws.com/

The plug-in has not been exhaustively tested. It works for me on Eclipse 3.7 Indigo. I have no idea what would happen -- and take no responsibility for what *might* happen -- if you try to install it on an OS that is not Mac OS.
