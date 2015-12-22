---
layout: post
title:  "Setting expires header in Nginx"
date:   2015-12-22 22:13:35
---

Or how to leverage browser cashing if you are using Nginx.

Leveraging browser cache can greatly improve your webapp's startup time. The browser doesn't need to make requests to server for things like static assets, which aren't likely to change every day. According to Google,

> Setting an expiry date or a maximum age in the HTTP headers for static resources instructs the browser to load previously downloaded resources from local disk rather than over the network. - [https://developers.google.com/speed/docs/insights/LeverageBrowserCaching](https://developers.google.com/speed/docs/insights/LeverageBrowserCaching)

So all we have to do is to set a header named 'Cache-Control' in our HTTP responses for static assets. Here is how you do it in Nginx -

Suppose you are serving url /static from a directory /srv/www-data/static. Then your location block would look like

    location /static/ {
        root /srv/www-data;
        expires 1y;
    }
 
This sets the Cache-Control header for all files in static folder to expire in one year.  Or to setup expiry header for particular files, you can do this

    location ~* \.(?:css|gif|jpe?g|png)$ {
        expires max;
    }

~* tells nginx to perform a case-insensitive regular expression match. For example, it can match /gravatar.png or /COMPUTER.JPG. "expires max" set the expire value to maximum, which is 10 years according to the doc.  You can also set the expire time in minutes, hours, days etc - [http://nginx.org/en/docs/http/ngx_http_headers_module.html#expires](http://nginx.org/en/docs/http/ngx_http_headers_module.html#expires)

Having this header in the responses will help the browser to determine if it needs to make a request to server or just reuse the file it has already stored in the cache. Not only this saves load time of our apps, but saves bandwidth and rendering time too.

Etag
----

'Cache-Control' is one way to enable browser cache in HTTP 1.1. The other one is an [Etag header](http://nginx.org/en/docs/http/ngx_http_core_module.html#etag). Etag is simply a token that identifies version of a static file served by Nginx. For example, see the headers sent MaxCDN's servers for bootstrap below

    $ curl -I https://maxcdn.bootstrapcdn.com/bootstrap/3.3.6/css/bootstrap.min.css
    HTTP/1.1 200 OK
    Date: Tue, 22 Dec 2015 17:42:11 GMT
    Content-Type: text/css
    Content-Length: 121260
    Connection: keep-alive
    Last-Modified: Tue, 24 Nov 2015 19:49:46 GMT
    ETag: "2f624089c65f12185e79925bc5a7fc42"
    Server: NetDNA-cache/2.2
    Expires: Fri, 16 Dec 2016 17:42:11 GMT
    Cache-Control: max-age=31104000
    Vary: Accept-Encoding
    Access-Control-Allow-Origin: *
    X-Cache: HIT
    Accept-Ranges: bytes

Notice the etag line - ETag: "2f624089c65f12185e79925bc5a7fc42".  If etag is enabled, browsers will send etag in the header for subsequent requests so that the server can determine to send the file (if updated) or tell browser to use load the file from its cache (usually indicated by 200 and 304 status codes respectively). Looking at MaxCDN's response, etag returned is simply MD5 hash of the file.

'Cache-Control' vs Etag
-------------------------------
So which should we use? Etag has a disadvantage that the browser has to always make a request to the server to see if the file has changed or not. However, for expire headers, browsers wont make a request until the expire date is close or passed. The good old [Yahoo Web Performance Best Practices and Rules](http://yslow.org/) suggests to [avoid etags](https://developer.yahoo.com/performance/rules.html#etags=). However the guide talks about Apache and IIS mostly. I usually setup both expire headers and etags for every app which has a single server setup. This way both the server and client have better ways to decide to use browser cache or not.h

Invalidating caches/responses
----------------------------------------
If one of the static assets served using the 'Cache-Control' is changed, how can we update (or invalidate) the cache so that browser will request for an updated file? For example, assume that we've sent a CSS file earlier with an expiry header for 1 year and we just committed a change in that CSS file. To force the browser to get an updated copy, all we need to do is change the URL of the CSS file. For example, we can change /css/styles.css to /css/styles.v1.css and the browser will get our updated file.

Tuning your webserver for maximum performance is both fun and rewarding. Here are a couple of links for you to get started -

[Yslow page](http://yslow.org/) (Scroll down to Web Performance Best Practices and Rules)

[High Performance Web Sites](http://www.amazon.com/gp/product/0596529309) You can read all 14 rules described in the book [online too](http://stevesouders.com/hpws/).
