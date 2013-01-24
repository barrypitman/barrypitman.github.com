---
layout: post
title: "CSS Background Image to DataURI translation at runtime"
description: "Replacing background-images with data URIs at runtime"
category: css
tags: [css, datauri, runtime]
---
{% include JB/setup %}


## Update: 24/01/2013:
[wro4j](http://code.google.com/p/wro4j/) (Web Resource Optimizer for Java) has a
"[CssDataUriProcessor"](http://code.google.com/p/wro4j/wiki/Base64DataUriSupport) that performs pretty much the same
function as the filter described here. I would suggest that you checkout wro4j's implementation, not least because its
a great project and well supported. Plus, it goes one better by providing a
["FallbackCssDataUriProcessor"](http://code.google.com/p/wro4j/wiki/FallbackCssDataUriProcessor), which has a
fallback for older IE versions, so you can use the same stylesheet.

## Introduction
[Data URI's](http://en.wikipedia.org/wiki/Data_URI_scheme) are a method of embedding inline data in web page responses.
For instance, using data URIs, a background image can be embedded within a
CSS file, reducing the number of requests that must be made to the originating server.
According to Yahoo’s [YSlow](http://developer.yahoo.com/yslow/) tool, minimizing HTTP requests is number one on the list of
'Web Performance Best Practices and Rules'.

A data URI is a string representation of a file’s contents (base64 encoded), with the following format:

    data:[<MIME-type>][;charset=<encoding>][;base64],<data>

e.g. a small red dot in an <img> tag:
    <img src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA
    AAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO
    9TXL0Y4OHwAAAABJRU5ErkJggg==" alt="Red dot"/>
as opposed to
    <img src="../images/red_dot.png" alt'"Red dot"/>

Using base64 encoding will increase the size of the image by 1/3, but that increase can be offset by
gzipping the response (and you’ll be making fewer requests). [Wikipedia](http://en.wikipedia.org/wiki/Data_URI_scheme)
has a good section on the pros/cons
of using data URI’s. One serious disadvantage is that older browsers, e.g. IE < 7, don’t support data URIs

## Data URIs as background images
Data URIs are normally embedded in files which are meant to be cached by the brower. One common use-case
is replacing background images references in CSS files to data URIs, e.g.

    .warningImage {
        background: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAIAAAD8GO2jAAAACXBIWXMAAC4jAA...=') no-repeat;
    }
instead of:

    .warningImage {
        background: url('../images/icon_warning_32x32.png') no-repeat;
    }

This method has similar benefits to using ['image sprites'](http://www.alistapart.com/articles/sprites),
but also has [several advantages](http://www.nczonline.net/blog/2010/07/06/data-uris-make-css-sprites-obsolete/) over them.
In many cases, background images are small icons, background gradients etc. which are well suited
to be replaced with data URIs. I noticed that Google is using both techniques on their search results
pages; the Google logo and other icons are served as an image sprite, while dynamic images/thumbnails
are served as data-uri images inline with the content.

The primary advantage of data URI background images over image sprites is that they are easier to maintain.
Still, I’d argue that maintaining them won’t be breeze; you’ll have to:

+ maintain two copies of the css file (one with links, one with data URIs)
+ a copy of the image
+ keep them in sync over time
+ serve the data URI version to browsers which support them.

There are several online tools which can assist by converting your images to data URIs,
but wouldn’t it be nice to be able to automatically convert background images at runtime and
serve the right version automatically?

## Runtime replacement of background-images with data URIs
I have implemented a Java servlet filter
([source available on GitHub](https://github.com/barrypitman/CSS-data-URI-substitution-filter))
which dynamically 'includes' background images that are referenced from CSS files as data URIs.
Using this filter alleviates the burden of maintaining two separate sets of CSS files; you only
need one have one CSS file and the filter will include background images if the user agent supports
them at runtime. Simply put, it includes/inlines background images by performing server-side includes
on the image references at runtime, using

    httpServletRequest.getRequestDispatcher(backgroundImageUrl).include(httpServletRequest, wrappedResponse);

Obviously the filter should only be mapped to CSS files that you’d like to perform this tranformation on.

**The filter operates as follows:**

1. Check the the user agent supports data URIs. If not, skip filter.
2. Wrap the response to the CSS file request with a custom ServletResponseWrapper which captures the response in a
CharArrayWriter instead of streaming it back to the client (so we can modify it later).
3. Pass the request along the filter chain, i.e. `filterChain.doFilter(httpServletRequest, wrappedResponse);`
4. Extract the response as a string. At this point we have the contents of the CSS file as a string.
5. Using regex, find references to background images within the CSS contents.
6. For each reference to a background image url (e.g. '../images/button.gif'):
  1. Wrap the response with a new custom ServletResponseWrapper which captures the response in a new ByteArrayOutputStream
  2. Use the request dispatcher to 'include' the referenced image, i.e.
    `request.getRequestDispatcher(backgroundImageUrl).include(httpServletRequest, imageResponseWrapper);`
  3. Extract the image contents as a byte[] from the wrapped response
  4. Using the filename and byte[], convert the image to a data URI representation
  5. Replace the reference to backgroud image url in the CSS file contents (from top-level step 4) with its data URI
  representation.
  6. Note: if a step above fails (e.g. 404 for the image), we leave the original reference intact.
7. Write the modified CSS file contents back to the client.

Obviously the regex used to match extract the image URLs from the contents of the CSS file is an
important part of this process. Getting it wrong could cause one to mess up the returned CSS
content badly! The patten which I have so far (which could probably be improved upon) is:

    \bbackground(-image)?:\s*url\(['|"]?(.*?)['|"]?\)

In addition to the pattern above, the filter will ignore image URLs that appear to reference
external images (e.g. on other sites) as well as images that are already in data URI format.

I have tested this filter on some fairly large CSS files, including a  minized version of the
jQuery UI stylesheet, and it was able to replace all background image references with their data
URI representation without problems. It has been tested in Jetty and Tomcat.

## Notes
1. The filter could easily be extended to be able to include other types of content
2. The filter assumes that the background url is the first listed style attribute,
so it will replace `background: url(img.gif) #FFF` but not `background: #FFF url(img.gif)`
3. It can only include images that can be served by the application; i.e. references to images on external
domains will be ignored.
4. The filter can be configured to not replace references to background images whose size is larger than a
certain value, e.g. 32KB
5. Because the filter dynamically modifies the response by including images, changes to the actual images
will not cause the last-modified headers of the CSS file to be updated; you'll need to `touch` the CSS file as
well if you want client to fetch a new version after modifying an image.
6. Be aware that embedding large background images in style sheets can have an adverse effect on page
rendering; essentially your images will be loaded much sooner that if you left them as URL references.
See this article for a great explanation. Obviously the idea is that you cache the larger CSS file,
so as long as the CSS file is already in the cache, you won’t have that problem.
