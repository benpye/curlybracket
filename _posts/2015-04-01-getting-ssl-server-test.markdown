---
layout: post
title:  "Getting an A+ on the SSL server test"
date:   2015-04-01 16:47:00 +0000
categories: admin
---
[![A+ Score]({{ site.baseurl }}/images/hero/ssl-test-a+.png)](https://www.ssllabs.com/ssltest/analyze.html?d=curlybracket.co.uk)

Setting up this website has always been more of a learning exercise for myself than trying to provide a useful resource to anybody. I'd be very happy if anyone finds any value in what I write, but in general I do it more to know how to configure a web server correctly. Recently I've tried to increase the scope of my knowledge, and now I have both automated Amazon S3 backups (which I will write about at a later date), and this server also scores A+ on the [Qualys SSL test](https://www.ssllabs.com/ssltest/), the highest rank achievable.

<!--more-->

I have never wanted to spend a great deal in setting up this website, and so I did not use an expensive wildcard certificate as I would prefer. This does limit the website to clients that support SNI as I use both www.curlybracket.co.uk and static.curlybracket.co.uk . All the certificates I use are provided by [StartSSL](https://www.startssl.com/). Up until recently StartSSL were issuing SHA-1 certificates which are now considered weak, however they now issue SHA-256 certificates that should be suitable for the forseeable future. Currently static.curlybracket.co.uk is stuck on a weak certificate until it expires and I can get a new certificate.

All content on this server is served using Nginx. Nginx is very good in allowing you to configure a secure website. From the default configuration, I have also enabled OCSP, HSTS, session resumption and SPDY 3.1, disabled as many weak ciphers as possible without severely limiting the clients that can connect and ensured that Nginx is using a sufficiently large prime number for forward secrecy.

OCSP
----
OCSP means that the server tells the client if any certificates for the domain are revoked by a signed revocation list, cached from the CA. This means that the client does not have to contact the certificate authority to ensure that the certificate used by the server is not revoked.  The following must be added to your Nginx configuration.

```
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certifcate <path to ca certifcate>;
resolver <dns server>;
```

We must provide a DNS server so that the server can fetch the revocation list from the CA if it is given using a domain name. Strictly only the `ssl_stapling on` is required, the rest mean that Nginx will verify the revocation list before sending it to the client.

HSTS
----
HSTS or HTTP strict transport security is enabled by adding to the servers response header. This means that the client will only connect to your server and any subdomains using HTTPS and will not even attempt a HTTP connection. Obviously you should only enable this if you are sure you will keep HTTPS enabled. In addition Chrome and Firefox (at least) have a list of domains within the browser that they know to have HSTS enabled for and you can request to be added to the list [here](https://hstspreload.appspot.com/). To be added to the list and to enable HSTS you should add the following to your servers configuration. `max-age` can be changed, `preload` is non standard and is used to ensure you wish to be added to the preload list and `includeSubDomains` means that the setting doesn't just apply to the top level domain.

```
add_header Strict-Transport-Security "max-age=31536000; preload; includeSubDomains";
```

Session Resumption
------------------
Session resumption is easy to enable in Nginx and only requires enabling the session cache by the following.

```
ssl_session_cache shared:ssl_session_cache:10m;
```

SPDY
----
Not required for an A+ score on the SSL test however useful anyway. This will soon be deprecated and replaced by HTTP/2.0 but until Nginx supports HTTP/2.0 this is the best I can do. SPDY is enabled in Nginx by adding `spdy` to your `listen` statement in your sever configuration assuming that HTTPS already works.

Secure Ciphers
--------------
Nginx by default will enable many ciphers, some of which will be weak and so should be disabled. For more detail on what ciphers are recommended for which usecase you should see [Mozilla's recommendation](https://wiki.mozilla.org/Security/Server_Side_TLS). I have used the intermediate list for my website as this means that most clients can connect, and security is maintained. In Nginx this means the following settings.

```
ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
ssl_prefer_server_ciphers on;
ssl_ciphers "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA";
```

Large Secret
------------
Nginx by default will use a small 1024 bit secret for forward secrecy, this can be considered weak. To work around this we must supply our own prime. To generate the prime we can use OpenSSL

```
openssl dhparam -rand – 4096
```
And then we must configure Nginx to use it

```
ssl_dhparam <path to dhparam>;
```

Hopefully if you can follow the above, you will also be able to achieve an A+ score on the SSL test 
