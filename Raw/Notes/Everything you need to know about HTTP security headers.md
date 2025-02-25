---
title: Everything you need to know about HTTP security headers
source: https://blog.appcanary.com/2017/http-security-headers.html
clipped: 2023-09-04
published: 
category: development
tags:
  - network
  - http
read: false
---

Some physicists 28 years ago needed a way to easily share experimental data and thus the web was born. This was generally considered to be a good move. Unfortunately, everything physicists touch — from trigonometry to the strong nuclear force — eventually becomes weaponized and so too has the Hypertext Transfer Protocol.

What can be attacked must be defended, and since tradition requires all security features to be a bolted-on afterthought, things… got a little complicated.

This article explains what secure headers are and how to implement these headers in Rails, Django, Express.js, Go, Nginx, Apache and Varnish.

Please note that some headers may be best configured in on your HTTP servers, while others should be set on the application layer. Use your own discretion here. You can test how well you’re doing with Mozilla’s [Observatory](https://observatory.mozilla.org/analyze.html?host=appcanary.com).

Did we get anything wrong? Contact us at [hello@appcanary.com](mailto:hello@appcanary.com).

## HTTP Security Headers

-   [X-XSS-Protection](#x-xss-protection)
    -   [Why?](#x-xss-protection/why)
    -   [Should I use it?](#x-xss-protection/should)
    -   [How?](#x-xss-protection/how)
    -   [I want to know more](#x-xss-protection/more)
-   [Content Security Policy](#csp)
    -   [Why?](#csp/why)
    -   [Should I use it?](#csp/should)
    -   [How?](#csp/how)
    -   [I want to know more](#csp/more)
-   [HTTP Strict Transport Security (HSTS)](#hsts)
    -   [Why?](#hsts/why)
    -   [Should I use it?](#hsts/should)
    -   [How?](#hsts/how)
    -   [I want to know more](#hsts/more)
-   [HTTP Public Key Pinning (HPKP)](#hpkp)
    -   [Why?](#hpkp/why)
    -   [Should I use it?](#hpkp/should)
    -   [How?](#hpkp/how)
    -   [I want to know more](#hpkp/more)
-   [X-Frame-Options](#x-frame-options)
    -   [Why?](#x-frame-options/why)
    -   [Should I use it?](#x-frame-options/should)
    -   [How?](#x-frame-options/how)
    -   [I want to know more](#x-frame-options/more)
-   [X-Content-Type-Options](#x-content-type-options)
    -   [Why?](#x-content-type-options/why)
    -   [Should I use it?](#x-content-type-options/should)
    -   [How?](#x-content-type-options/how)
    -   [I want to know more](#x-content-type-options/more)
-   [Referrer-Policy](#referrer-policy)
    -   [Why?](#referrer-policy/why)
    -   [Should I use it?](#referrer-policy/should)
    -   [How?](#referrer-policy/how)
    -   [I want to know more](#referrer-policy/more)
-   [Cookie Options](#cookie-options)
    -   [Why?](#cookie-options/why)
    -   [Should I use it?](#cookie-options/should)
    -   [How?](#cookie-options/how)
    -   [I want to know more](#cookie-options/more)

---

## X-XSS-Protection

```
X-XSS-Protection: 0;
X-XSS-Protection: 1;
X-XSS-Protection: 1; mode=block
```

### Why?

Cross Site Scripting, commonly abbreviated XSS, is an attack where the attacker causes a page to load some malicious javascript. `X-XSS-Protection` is a feature in Chrome and Internet Explorer that is designed to protect against “reflected” XSS attacks — where an attacker is sending the malicious payload as part of the request[1](#fn1).

`X-XSS-Protection: 0` turns it off.  
`X-XSS-Protection: 1` will filter out scripts that came from the request - but will still render the page  
`X-XSS-Protection: 1; mode=block` when triggered, will block the whole page from being rendered.

### Should I use it?

Yes. Set `X-XSS-Protection: 1; mode=block`. The “filter bad scripts” mechanism is problematic; see [here](http://blog.innerht.ml/the-misunderstood-x-xss-protection/) for why.

### How?

Platform

What do I do?

Rails 4 and 5

On by default

Django

`SECURE_BROWSER_XSS_FILTER = True`

Express.js

Use [helmet](https://helmetjs.github.io/docs/xss-filter/)

Go

Use [unrolled/secure](https://github.com/unrolled/secure)

Nginx

`add_header X-XSS-Protection "1; mode=block";`

Apache

`Header always set X-XSS-Protection "1; mode=block"`

Varnish

`set resp.http.X-XSS-Protection = "1; mode=block";`

### I want to know more

[X-XSS-Protection - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection)

---

## Content Security Policy

```
Content-Security-Policy: <policy>
```

### Why?

Content Security Policy can be thought of as much more advanced version of the `X-XSS-Protection` header above. While `X-XSS-Protection` will block scripts that come from the request, it’s not going to stop an XSS attack that involves storing a malicious script on your server or loading an external resource with a malicious script in it.

CSP gives you a language to define where the browser can load resources from. You can white list origins for scripts, images, fonts, stylesheets, etc in a very granular manner. You can also compare any loaded content against a hash or signature.

### Should I use it?

Yes. It won’t prevent all XSS attacks, but it’s a significant mitigation against their impact, and an important aspect of defense-in-depth. That said, it can be hard to implement. If you’re an intrepid reader and went ahead and checked the headers [appcanary.com](https://appcanary.com/) returns[2](#fn2), you’ll see that we don’t have CSP implemented yet. There are some rails development plugins we’re using that are holding us back from a CSP implementation that will have an actually security impact. We’re working on it, and will write about it in the next instalment!

### How?

Writing a CSP policy can be challenging. See [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy) for a list of all the directives you can employ. A good place to start is [here](https://csp.withgoogle.com/docs/adopting-csp.html).

Platform

What do I do?

Rails 4 and 5

Use [secureheaders](https://github.com/twitter/secureheaders)

Django

Use [django-csp](https://github.com/mozilla/django-csp)

Express.js

Use [helmet/csp](https://github.com/helmetjs/csp)

Go

Use [unrolled/secure](https://github.com/unrolled/secure)

Nginx

`add_header Content-Security-Policy "<policy>";`

Apache

`Header always set Content-Security-Policy "<policy>"`

Varnish

`set resp.http.Content-Security-Policy = "<policy>";`

### I want to know more

-   [Content-Security-Policy - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy)
-   [CSP Quick Reference Guide](https://content-security-policy.com/)
-   [Google’s CSP Guide](https://csp.withgoogle.com/docs/index.html)

---

## HTTP Strict Transport Security (HSTS)

```
Strict-Transport-Security: max-age=<expire-time>
Strict-Transport-Security: max-age=<expire-time>; includeSubDomains
Strict-Transport-Security: max-age=<expire-time>; preload
```

### Why?

When we want to securely communicate with someone, we face two problems. The first problem is privacy; we want to make sure the messages we send can only be read by the recipient, and no one else. The other problem is that of authentication: how do we know the recipient is who they say they are?

HTTPS solves the first problem with encryption, though it has some major issues with authentication (more on this later, see [Public Key Pinning](#http-public-key-pinning-hpkp)). The HSTS header solves the meta-problem: how do you know if the person you’re talking to actually supports encryption?

HSTS mitigates an attack called [sslstrip](https://moxie.org/software/sslstrip/). Suppose you’re using a hostile network, where a malicious attacker controls the wifi router. The attacker can disable encryption between you and the websites you’re browsing. Even if the site you’re accessing is only available over HTTPS, the attacker can man-in-the-middle the HTTP traffic and make it look like the site works over unencrypted HTTP. No need for SSL certs, just disable the encryption.

Enter the HSTS. The `Strict-Transport-Security` header solves this by letting your browser know that it must always use encryption with your site. As long as your browser has seen an HSTS header — and it hasn’t expired — it will not access the site unencrypted, and will error out if it’s not available over HTTPS.

### Should I use it?

Yes. Your app is only available over HTTPS, right? Trying to browse over regular old HTTP will redirect to the secure site, right? (Hint: Use [letsencrypt](https://letsencrypt.org/) if you want to avoid the racket that are commercial certificate authorities.)

The one downside of the HSTS header is that it allows for a [clever technique](http://www.radicalresearch.co.uk/lab/hstssupercookies) to create supercookies that can fingerprint your users. As a website operator, you probably already track your users somewhat - but try to only use HSTS for good and not for supercookies.

### How?

The two options are

-   `includeSubDomains` - HSTS applies to subdomains
-   `preload` - Google maintains a [service](https://hstspreload.appspot.com/) that hardcodes[3](#fn3) your site as being HTTPS only into browsers. This way, a user doesn’t even have to visit your site: their browser already knows it should reject unencrypted connections. Getting off that list is hard, by the way, so only turn it on if you know you can support HTTPS forever on all your subdomains.

Platform

What do I do?

Rails 4

`config.force_ssl = true`  
Does not include subdomains by default. To set it:  
`config.ssl_options = { hsts: { subdomains: true } }`

Rails 5

`config.force_ssl = true`

Django

`SECURE_HSTS_SECONDS = 31536000`  
`SECURE_HSTS_INCLUDE_SUBDOMAINS = True`

Express.js

Use [helmet](https://helmetjs.github.io/docs/hsts/)

Go

Use [unrolled/secure](https://github.com/unrolled/secure)

Nginx

`add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; ";`

Apache

`Header always set Strict-Transport-Security "max-age=31536000; includeSubdomains;`

Varnish

`set resp.http.Strict-Transport-Security = "max-age=31536000; includeSubdomains; ";`

### I want to know more

-   [RFC 6797](https://tools.ietf.org/html/rfc6797)
-   [Strict-Transport-Security - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)

---

## HTTP Public Key Pinning (HPKP)

```
Public-Key-Pins: pin-sha256=<base64==>; max-age=<expireTime>;
Public-Key-Pins: pin-sha256=<base64==>; max-age=<expireTime>; includeSubDomains
Public-Key-Pins: pin-sha256=<base64==>; max-age=<expireTime>; report-uri=<reportURI>
```

### Why?

The HSTS header described above was designed to ensure that all connections to your website are encrypted. However, nowhere does it specify what key to use!

Trust on the web is based on the certificate authority (CA) model. Your browser and operating system ship with the public keys of some trusted certificate authorities which are usually specialized companies and/or nation states. When a CA issues you a certificate for a given domain that means anyone who trusts that CA will automatically trust the SSL traffic you encrypt using that certificate. The CAs are responsible for verifying that you actually own a domain (this can be anything from sending an email, to asking you to host a file, to investigating your company).

Two CAs can issue a certificate for the same domain to two different people, and browsers will trust both. This creates a problem, especially since CAs [can be and are](https://technet.microsoft.com/library/security/2524375) compromised. This allows attackers to MiTM any domain they want, even if that domain uses SSL & HSTS!

The HPKP header tries to mitigate this. This header lets you to “pin” a certificate. When a browser sees the header for the first time, it will save the certificate. For every request up to `max-age`, the browser will fail unless at least one certificate in the chain sent from the server has a fingerprint that was pinned.

This means that you can pin to the CA or a intermediate certificate along with the leaf in order to not shoot yourself in the foot (more on this later).

Much like HSTS above, the HPKP header also has some privacy implications. These were laid out in the [RFC](https://tools.ietf.org/html/rfc7469#section-5) itself.

### Should I use it?

Probably not.

HPKP is a very very sharp knife. Consider this: if you pin to the wrong certificate, or you lose your keys, or something else goes wrong, you’ve locked your users out of your site. All you can do is wait for the pin to expire.

This [article](https://blog.qualys.com/ssllabs/2016/09/06/is-http-public-key-pinning-dead) lays out the case against it, and includes a fun way for attackers to use HPKP to hold their victims ransom.

One alternative is using the `Public-Key-Pins-Report-Only` header, which will just report that something went wrong, but not lock anyone out. This allows you to at least know your users are being MiTMed with fake certificates.

### How?

The two options are

-   `includeSubDomains` - HPKP applies to subdomains
-   `report-uri` - Inavlid attempts will be reported here

You have to generate a base64 encoded fingerprint for the key you pin to, and you **have** to use a backup key. Check [this guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Public_Key_Pinning#Extracting_the_Base64_encoded_public_key_information) for how to do it.

Platform

What do I do?

Rails 4 and 5

Use [secureheaders](https://github.com/twitter/secureheaders/blob/master/docs/HPKP.md)

Django

Write custom middleware

Express.js

Use [helmet](https://helmetjs.github.io/docs/hpkp/)

Go

Use [unrolled/secure](https://github.com/unrolled/secure)

Nginx

`add_header Public-Key-Pins 'pin-sha256="<primary>"; pin-sha256="<backup>"; max-age=5184000; includeSubDomains';`

Apache

`Header always set Public-Key-Pins 'pin-sha256="<primary>"; pin-sha256="<backup>"; max-age=5184000; includeSubDomains';`

Varnish

`set resp.http.Public-Key-Pins = "pin-sha256="<primary>"; pin-sha256="<backup>"; max-age=5184000; includeSubDomains";`

### I want to know more

-   [RFC 7469](https://tools.ietf.org/html/rfc7469)
-   [HTTP Public Key Pinning (HPKP) - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Public_Key_Pinning)

---

## X-Frame-Options

```
X-Frame-Options: DENY
X-Frame-Options: SAMEORIGIN
X-Frame-Options: ALLOW-FROM https://example.com/
```

### Why?

Before we started giving dumb names to vulnerabilities, we used to give dumb names to hacking techniques. “Clickjacking” is one of those dumb names.

The idea goes like this: you create an invisible iframe, place it in focus and route user input into it. As an attacker, you can then trick people into playing a browser-based game while their clicks are being registered by a hidden iframe displaying twitter - forcing them to non-consensually retweet all of your tweets.

It sounds dumb, but it’s an effective attack.

### Should I use it?

Yes. Your app is a beautiful snowflake. Do you really want some [genius](https://techcrunch.com/2015/04/08/annotate-this/) shoving it into an iframe so they can vandalize it?

### How?

`X-Frame-Options` has three modes, which are pretty self explanatory.

-   `DENY` - No one can put this page in an iframe
-   `SAMEORIGIN` - The page can only be displayed in an iframe by someone on the same origin.
-   `ALLOW-FROM` - Specify a specific url that can put the page in an iframe

One thing to remember is that you can stack iframes as deep as you want, and in that case, the behavior of `SAMEORIGIN` and `ALLOW-FROM` isn’t [specified](https://tools.ietf.org/html/rfc7034#section-2.3.2.2). That is, if we have a triple-decker iframe sandwich and the innermost iframe has `SAMEORIGIN`, do we care about the origin of the iframe around it, or the topmost one on the page? ¯\\\_(ツ)\_/¯.

Platform

What do I do?

Rails 4 and 5

`SAMEORIGIN` is set by default.

To set `DENY`:  
`config.action_dispatch.default_headers['X-Frame-Options'] = "DENY"`

Django

`MIDDLEWARE = [ ... 'django.middleware.clickjacking.XFrameOptionsMiddleware', ... ]`  
This defaults to `SAMEORIGIN`.

To set `DENY`: `X_FRAME_OPTIONS = 'DENY'`

Express.js

Use [helmet](https://helmetjs.github.io/docs/frameguard/)

Go

Use [unrolled/secure](https://github.com/unrolled/secure)

Nginx

`add_header X-Frame-Options "deny";`

Apache

`Header always set X-Frame-Options "deny"`

Varnish

`set resp.http.X-Frame-Options = "deny";`

### I want to know more

-   [RFC 7034](https://tools.ietf.org/html/rfc7034)
-   [X-Frame-Options - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options)

---

## X-Content-Type-Options

```
X-Content-Type-Options: nosniff;
```

### Why?

The problem this header solves is called “MIME sniffing”, which is actually a browser “feature”.

In theory, every time your server responds to a request it is supposed to set a `Content-Type` header in order to tell the browser if it’s getting some HTML, a cat gif, or a Flash cartoon from 2008. Unfortunately, the web has always been broken and has never really followed a spec for anything; back in the day lots of people didn’t bother to set the content type header properly.

As a result, browser vendors decided they should be really helpful and try to infer the content type by inspecting the content itself while completely ignore the content type header. If it looks like a gif, display a gif!, even though the content type is `text/html`. Likewise, if it looks like we got some HTML, we should render it as such even if the server said it’s a gif.

This is great, except when you’re running a photo-sharing site, and users can upload photos that look like HTML with javascript included, and suddenly you have a stored XSS attack on your hand.

The `X-Content-Type-Options` headers exist to tell the browser to shut up and set the damn content type to what I tell you, thank you.

### Should I use it?

Yes, just make sure to set your content types correctly.

### How?

Platform

What do I do?

Rails 4 and 5

On by default

Django

`SECURE_CONTENT_TYPE_NOSNIFF = True`

Express.js

Use [helmet](https://helmetjs.github.io/docs/dont-sniff-mimetype/)

Go

Use [unrolled/secure](https://github.com/unrolled/secure)

Nginx

`add_header X-Content-Type-Options nosniff;`

Apache

`Header always set X-Content-Type-Options nosniff`

Varnish

`set resp.http.X-Content-Type-Options = "nosniff";`

### I want to know more

-   [X-Content-Type-Options - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options)

---

## Referrer-Policy

```
Referrer-Policy: "no-referrer" 
Referrer-Policy: "no-referrer-when-downgrade" 
Referrer-Policy: "origin" 
Referrer-Policy: "origin-when-cross-origin"
Referrer-Policy: "same-origin" 
Referrer-Policy: "strict-origin" 
Referrer-Policy: "strict-origin-when-cross-origin" 
Referrer-Policy: "unsafe-url"
```

### Why?

Ah, the `Referer` header. Great for analytics, bad for your users’ privacy. At some point the web got woke and decided that maybe it wasn’t a good idea to send it all the time. And while we’re at it, let’s spell “Referrer” correctly[4](#fn4).

The `Referrer-Policy` header allows you to specify when the browser will set a `Referer` header.

### Should I use it?

It’s up to you, but it’s probably a good idea. If you don’t care about your users’ privacy, think of it as a way to keep your sweet sweet analytics to yourself and out of your competitors’ grubby hands.

Set `Referrer-Policy: "no-referrer"`

### How?

Platform

What do I do?

Rails 4 and 5

Use [secureheaders](https://github.com/twitter/secureheaders)

Django

Write custom middleware

Express.js

Use [helmet](https://helmetjs.github.io/docs/referrer-policy/)

Go

Write custom middleware

Nginx

`add_header Referrer-Policy "no-referrer";`

Apache

`Header always set Referrer-Policy "no-referrer"`

Varnish

`set resp.http.Referrer-Policy = "no-referrer";`

### I want to know more

-   [Referrer Policy - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy)

---

## Cookie Options

```
Set-Cookie: <key>=<value>; Expires=<expiryDate>; Secure; HttpOnly; SameSite=strict
```

### Why?

This isn’t a security header per se, but there are three different options for cookies that you should be aware of.

-   Cookies marked as `Secure` will only be served over HTTPS. This prevents someone from reading the cookies in a MiTM attack where they can force the browser to visit a given page.
    
-   `HttpOnly` is a misnomer, and has nothing to do with HTTPS (unlike `Secure` above). Cookies marked as `HttpOnly` can not be accessed from within javascript. So if there is an XSS flaw, the attacker can’t immediately steal the cookies.
    
-   `SameSite` helps defend against Cross-Origin Request Forgery (CSRF) attacks. This is an attack where a different website the user may be visiting inadvertently tricks them into making a request against your site, i.e. by including an image to make a GET request, or using javascript to submit a form for a POST request. Generally, people defend against this using [CSRF tokens](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet). A cookie marked as `SameSite` won’t be sent to a different site.
    

It has two modes, lax and strict. Lax mode allows the cookie to be sent in a top-level context for GET requests (i.e. if you clicked a link). Strict doesn’t send any third-party cookies.

### Should I use it?

You should absolutely set `Secure` and `HttpOnly`. Unfortunately, as of writing, SameSite cookies are [available](http://caniuse.com/#search=samesite) only in Chrome and Opera, so you may want to ignore them for now.

### How?

Platform

What do I do?

Rails 4 and 5

Secure and HttpOnly enabled by default. For SameSite, use [secureheaders](https://github.com/twitter/secureheaders)

Django

Session cookies are HttpOnly by default. To set secure: `SESSION_COOKIE_SECURE = True`.

Not sure about SameSite.

Express.js

`cookie: { secure: true, httpOnly: true, sameSite: true }`

Go

`http.Cookie{Name: "foo", Value: "bar", HttpOnly: true, Secure: true}`

For SameSite, see this [issue](https://github.com/golang/go/issues/15867).

Nginx

You probably won’t set session cookies in Nginx

Apache

You probably won’t set session cookies in Apache

### I want to know more

-   [Cookies - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#Secure_and_HttpOnly_cookies)

---

Thanks to [@wolever](https://twitter.com/wolever) for python advice.

Thanks to [Guillaume Quintard](https://twitter.com/therealgquintar) for Varnish comands.