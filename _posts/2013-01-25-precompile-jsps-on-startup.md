---
layout: post
title: "Pre-compiling all JSPs on demand"
description: ""
category: servlet
tags: [jsp, tomcat, jetty]
---
{% include JB/setup %}

## tl;dr:
This servlet filter allows you to pre-compile all of your JSP files with a single once-off 'trigger' request, useful as
a post-deploy step to give your application some snappiness at startup, or just to check that they do actually compile!
I tested on Tomcat and Jetty, but it should work for any JSP 2.1 - enabled container.

## Solution
I got the idea from [this post](http://pukkaone.github.com/2010/12/22/jsp-precompile-application-start.html):,
which uses a ServletContextListener and a JDK dynamic proxy to invoke each JSP in turn using the "precompilcation
protocol", which is a feature of the [JSP 2.1 specification](http://jsp.java.net/spec/jsp-2_1-fr-spec.pdf):

> A request to a JSP page that has a request parameter with name jsp_precompile
> is a precompilation request. The jsp_precompile parameter may have no value, or
> may have values true or false. In all cases, the request should not be delivered to the
> JSP page.
> The intention of the precompilation request is that of a suggestion to the JSP
> container to precompile the JSP page into its JSP page implementation class. The
> suggestion is conveyed by giving the parameter the value true or no value, but note
> that the request can be ignored.

I like that the described solution performs the compilation automatically at startup (without a trigger request),
but I found that it doesn't work in Jetty, because Jetty tries to convert the `javax.servlet.http.HttpServletRequest` to
`org.eclipse.jetty.server.Request` in `org.eclipse.jetty.server.RequestDispatcher`, which gives a NullPointerException
on this line:

    Request baseRequest=(request instanceof Request)?((Request)request):
                                AbstractHttpConnection.getCurrentConnection().getRequest();

## Servlet filter to the rescue!
So instead of trying to manufacture an HttpServletRequest, I figured it would be easier to use the real thing! I
created a servlet filter to look for requests containing a special request parameter, 'jsp_precompile_all'. Upon
receiving a request with this parameter,

 + Issue an 'include' to each of the *.jsp files in the application, with a query string of 'jsp_precompile'.
 + Catch any compilation exceptions, log them, and return a 500 response code if any occurred. This signals to the
 client making the request that there were JSP compilation failures and they should check the logs.
 + Include a mechanism to prevent this 'compilation' request from being processed more than once - recompiling all of
 your JSPs can take a while and load your server; we don't want any DoS attacks!

## Show me!

<script src="https://gist.github.com/4637213.js"></script>



