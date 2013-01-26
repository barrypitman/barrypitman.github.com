---
layout: post
title: "My experience with Turbolinks"
description: ""
category: javascript
tags: [servlet, turbolinks, rails]
---
{% include JB/setup %}

I tried out [Turbolinks](https://github.com/rails/turbolinks) in one of our applications and, [after all that has
been said](https://plus.google.com/106300407679257154689/posts/A65agXRynUn), I was pleasantly surprised by the fact
that it 'just worked'. It offered a noticeable speed improvement, and only required a couple of tweaks to our
javascript event-handling. That said, we were already setup for loading and initializing HTML fragments from the server,
so maybe our client-side code was better prepared for long-lived page scope. I would suggest that Turbolinks
is a quick and easy solution for improving the in-browser performance of mostly full-page HTML applications.

Even though it is incorporated into Rails, its pretty much a pure javascript solution, so its agnostic of the
server-side technology used. One limitation I found was that if you are using HTTP redirects (for instance
[post-redirect-get](http://en.wikipedia.org/wiki/Post/Redirect/Get)), then after a redirect,
the browser won't reflect the redirected URL.
[Apparently this is a limitation in XHR](https://github.com/rails/turbolinks/issues/22) - it has no way to detect
redirects. But still, I don't want the user copying and sharing the wrong URL, or having issues with page re-loading.

So Turbolinks includes a [hook](https://github.com/rails/turbolinks/pull/125) for you to pass the 'correct' URL as a
response header, `X-XHR-Current-Location`. All you need to do on the server side is to set the header.
I assume that somewhere Rails is configured to set that header. For Java webapps this is easily achieved with,
you guessed it, a servlet filter:.

<script src="https://gist.github.com/4637786.js"></script>

Another gotcha to look out for is the fact that you'll have to add 'data-no-turbolink' attributes to links
that can result in a file download. Otherwise I found that it would try replace the `<body>` with the binary contents
of the downloaded file - not pretty! I don't suppose there is any workaround for that - the client can't know that
following a link could trigger a file download?

