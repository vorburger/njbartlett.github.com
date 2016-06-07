---
layout: post
title: Using Let's Encrypt Certificates with OSGi HTTP Service
summary: 
tags: osgi http
date: 2016-06-07 12:00:00
comments: true
---

Rather than one of my usual polemics, this is a quickly-written practical post. I hope the information here is useful to somebody.

Currently I am setting up [effectiveosgi.com](https://effectiveosgi.com/), the website for my upcoming book "Effective OSGi". The site is a web app that will allow users to download preview PDFs, order print copies, and so on. Naturally I am developing it in Java and OSGi (in fact, many of the code samples in the book are based on the application that will be used to sell and deliver it).

Of course I want to follow best practices and use TLS (HTTPS), and this is much easier nowadays thanks to [Let's Encrypt](https://letsencrypt.org). However, it wasn't exactly obvious how to import the certicate provided by Let's Encrypt into a Java/OSGi application, so I'm documenting the steps that worked for me.

### Step One: Get the Certificate

This part I'm not going document in detail because it's more than adequately documented on the [Let's Encrypt - Getting Started](https://letsencrypt.org/getting-started/) page. I used the "standalone" method. The subsequent steps assume that you have successfully obtained your certificate files, which will be saved under the path `/etc/letsencrypt/live/[yourdomain]`. The files you will find there are `cert.pem`, `chain.pem`, `fullchain.pem` and `privkey.pem`.

### Step Two: Build Java Keystore

Note that this step is relevant for all Java developers, not just OSGi users.

Java applications cannot directly read PEM files; we need to generate a Java keystore file using the `keytool` program included with the JDK. Unfortunately `keytool` also can't read PEM files directly, so we first have to use `openssl` to convert them to PKCS12 format:

    openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem \
            -out cert_and_key.pkcs12 -name MyDomain \
            -CAfile chain.pem -caname root

You will be prompted to enter an export password. This will only be used in the next step, so it's fine to use just `password`. Now we have a file called `cert_and_key.pkcs12` which combines both the full certificate chain with the private signing key. Import this into a new keystore file as follows:

    keytool -importkeystore \
            -srckeystore cert_and_key.pkcs12 -srcstoretype PKCS12 -srcstorepass password \
            -destkeystore letsencrypt.jks -deststorepass XXXXXX -destkeypass XXXXXXX

You now have a keystore file named `letsencrypt.jks`. Keep the store and key passwords safe and secret!

### Step Three: Configure HTTP Service

This step is relevant to OSGi developers. I'll assume you are using the [Apache Felix HTTP Service](https://felix.apache.org/documentation/subprojects/apache-felix-http-service.html).

The HTTP Service is configured with the standard OSGi Configuration Admin. You need to create a configuration record with a PID of `org.apache.felix.http` with the following settings:

    org.apache.felix.https.enable true
    org.osgi.service.http.port.secure: 8443
    org.apache.felix.https.keystore: path/to/letsencrypt.jks
    org.apache.felix.https.keystore.password: XXXXXX
    org.apache.felix.https.keystore.key.password: XXXXXXX

This should be all you need to do get a green lock icon in your browser. Congratulations, you are now running a properly configured HTTPS server!

However... it's not very secure yet. Once your site is running, I **strongly** recommend testing your TLS implementation using [SSL Labs from Qualys](https://www.ssllabs.com/ssltest/). If you do this now, you will get at best a B grade.

### Step Four: Getting the A Grade

One of the security warnings reported by SSL Labs is due to [weak Diffie-Hellman groups](https://weakdh.org). I don't pretend to understand everything on the linked page, but fortunately the fix is easy: you just need to make sure your Java VM uses 2048-bit Diffie-Hellman keys by setting a system property. This can be specified on the command line as follows:

    java -Djdk.tls.ephemeralDHKeySize=2048 ...

To achieve [forward secrecy](https://blog.qualys.com/ssllabs/2013/06/25/ssl-labs-deploying-forward-secrecy) you also need to restrict the cipher suites to use only elliptic-curve algorithms. It's also a good idea to limit the server to 256-bit rather than 128-bit algos. This is done in the configuration admin record for the HTTP Service; just add the following:

    org.apache.felix.https.jetty.ciphersuites.included: \
        TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384, \
        TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA, \
        TLS_RSA_WITH_AES_256_CBC_SHA256, \
        TLS_RSA_WITH_AES_256_CBC_SHA

With these settings you should now be a grade A student:

<img width="600" src="/images/posts/ssl-grade-a.png">

Finally: you might be wondering how to get an elusive A+ grade on SSL Labs. Alas it doesn't currently seem to be possible in a Java-based server (though I may be wrong -- please let me know in the comments). The feature needed to unlock A+ is called [TLS_FALLBACK_SCSV](https://datatracker.ietf.org/doc/rfc7507/), and it requires an [API change](https://bugs.openjdk.java.net/browse/JDK-8061798). Hopefully it will be implemented in Java 9.