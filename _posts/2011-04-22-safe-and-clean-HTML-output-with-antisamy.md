---
layout: post
title: "Preventing XSS attacks with AntiSamy (Part 2)"
description: "Safe and clean HTML output with AntiSamy"
category: java
tags: [xss, antisamy]
---
{% include JB/setup %}

## Introduction

My [last blog post]({% post_url 2011-04-14-using-input-validation-XSS %}) discussed one method of protecting your web
application against Cross-Site Scripting (XSS) attacks; by filtering all user input with an
[AntiSamy servlet filter](https://github.com/barrypitman/antisamy-servlet-filter)
during request processing. In this post I’d like to discuss an alternative/complementary method involving AntiSamy;
filtering HTML-formatted content during response processing.

## Recap
To protect users from XSS attacks we should escape user-supplied inputs using e.g. `<c:out/>` tags in JSP files.
We can’t escape HTML-formatted content with ‘<c:out/>’ tags because we’d be escaping the formatting as well.
We need an intelligent library like AntiSamy to decide what formatting to allow and what to remove
(based on a well-defined policy).

When displaying user-supplied HTML-formatted content to a user, XSS attacks are not the only thing we need to
worry about. Imagine how this input: `</div></div>` could potentially mess up your layout! Fortunately AntiSamy
will also check for unclosed tags and remove/close them for you.

## Solution
The solution is implemented as a JSP custom tag, making it simple to re-use. Using it in a JSP file might look like this:

    <safeHtml:out text="${someBean.htmlTextToRender}"/>

We can also offer the developer the option of using a custom AntiSamy policy file:

<code id="gist-4557487" data-file="SafeHtmlTag.java"></code>

The tag implementation caches the loaded policy files internally as an optimization, and loads the policy
file called ‘antisamy-default.xml’ from the classpath if no custom policyFile is specified.
I’ve updated the source in the [GitHub project](https://github.com/barrypitman/antisamy-servlet-filter) from my
last post with this tag.

I hope someone finds this useful!
