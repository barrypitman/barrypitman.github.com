---
layout: post
title: "Preventing XSS attacks with AntiSamy (Part 1)"
description: "Using input validation to filter XSS in input parameters"
category: java
tags: [xss, antisamy]
---
{% include JB/setup %}

<p align="center">
  <img src="http://imgs.xkcd.com/comics/exploits_of_a_mom.png"/>
</p>


Cross Site Scripting (XSS) attacks are a well documented vulnerability affecting many web applications.
There has been a spate of high-profile XSS cases recently. Methods of protecting against these attacks
depend on whether you want to accept any HTML formatted text from your users or not:

1.  #### Escape all user-supplied data on rendering
    If you don’t plan on accepting HTML input, you can simply escape all user-supplied data.
    In a Java JSP application, this means outputting your text using the `<c:out>` tag or `${fn:escapeXml(text)}`
    function instead of just `${text}`. Once you’re in the habit of doing this, it might not be too bad,
    however forgetting just one instance will leave your application vulnerable to attack.
    Its easy to imagine a developer forgetting to escape some text while developing an email template.
2.  #### Filter out invalid/malicious user-supplied data on submit
    If you do plan on accepting HTML input, you need to decide what type of input is allowed.
    Many forums and applications using rich-text editors accept a limited subset of HTML from the user.
    Most likely you’ll want to exclude/filter `<script>` tags.

But there are many other tags/characters you would want to exclude too.
Just look at http://ha.ckers.org/xss.html to see some of the vast array of exploits,
and those are just the documented ones. Protecting against all possible attacks by exclusion is almost impossible.

The preferred method is to specify what content the user is allowed to submit
(a whitelist as opposed to a blacklist), and filter/purify everything else.
Writing such a library is tricky at best, so go with a recognised, widely used solution.
HTML Purifier for PHP and AntiSamy for Java/.NET are two such solutions.

## This is difficult!

Implementing either of the solutions above on a case-by-case basis would be a pain.
It gets in the way of what we really want to do, which is develop new features.
Ideally we would like a solution that protects the application and its users by default.
It would have been nice if the JSP default for outputting text with ${} was to escape it.
Then applications would be safe (to a large degree) from XSS attacks by default,
and we could un-escape text when we wanted to. Unfortunately that’s not the case.
Edit: here is an article describing how to override JSP EL expressions by default,
though I haven’t tried it yet.

## A solution
One solution is to implement a Java servlet filter which intercepts requests to the
server before they are processed, cleaning the submitted parameters using AntiSamy.
The application will only see the sanitized request parameters, so that the user-supplied
data is safe for re-display later. Assuming that you don’t already have malicious content
in your database, this approach should keep your users safe from XSS attacks, without you
needing to escape text every time you display it. Note however that for non-HTML formatted
text, you might as well escape user-supplied text as a precaution.
The advantage of using a filter is that it can very easily be applied to all requests and all
requests parameters, without your business logic having worry about cleaning submitted data.
The obvious disadvantage to using a filter-based approach is that it is more difficult to
cater for exceptions to the rule. AntiSamy policy files allow one to declare your own rules,
so you can be as strict or lax as you like. It’s difficult to imagine a scenario where a user submitting:

username: `<SCRIPT/XSS SRC="http://ha.ckers.org/xss.js"></SCRIPT>`

would be ok!
Another disadvantage is the processing overhead of passing every request parameter through AntiSamy.
Although AntiSamy is designed for speed, my experience is that passing parameters through the
filter costs approximately 0.3ms per parameter on my machine (for shortish strings). For a large form
that might add up to 20ms or more. Depending on your application that may or may not be a problem.

## Implementation
Here is my filter class, using a HttpServletRequestWrapper internally to override the `getParameter()`-type methods:

{% gist 4557334 AntiSamyFilter.java %}

### Notes:
+ This method will clean data submitted by regular form submissions from a browser.
  If your application has other methods of data-input, e.g. a SOAP web service,
  you will need to cater for that separately.

+ The filter will ‘modify’ the user-submitted data if it is deemed dangerous.
  This is something to bear in mind when you are debugging.
  The filter will at least log instances where inputs have been modified.

+ I haven’t tested it with file uploads so I’m not sure what effect, if any, that would have.

+ The implementation does not cache the filtered values, so calling request.getParameter(“name”)
  twice will cause the value to be cleaned twice. Caching could be implemented if required.

+ I’ve uploaded the source code as a maven project on GitHub. It includes a sample web application
  with simple JSP for testing the output of XSS hacks against the filter above, and a
  maven-jetty-plugin configuration to get going quickly.

+ This is an application-level filter. You might also want to use a firewall,
  e.g. mod_security for Apache for defence-in-depth.

### Useful Resources:
1. <http://code.google.com/p/doctype/wiki/ArticleIntroductionToXSS>
2. <http://blog.modsecurity.org/2008/07/do-you-know-how.html>
3. <http://ha.ckers.org/xss.html>

Please let me know if you find any issues, or have suggestions for improvement!
