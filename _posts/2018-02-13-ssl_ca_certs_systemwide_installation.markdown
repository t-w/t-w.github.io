---
layout: post
title: "SSL CA Certificates system-wide installation"
date: 2018-02-13
categories: ssl ca certificate
tags: ssl ca certificate installation system-wide
---
Using SSL (ie. OpenSSL) in your programs (or with tools like `curl`) with
services securing communication with certificates issued by some local
Certificate Authority requires providing these CA certificates (clearly
for validating the service that we are connecting to). Providing them
separatetly for each program is possible but kind of annoying, it might be
easier to install them once, in some system-wide way.
They will be also protected with sysadmin rights and cannot be replaced easily.

Seemingly obvious task is not that seemingly obviously anywhere explained,
so couple of notes that should basically solve the problem.

To make a CA certificate(s) available for program in the system (done on Debian,
should be the same / very similar on others):

1. Get the file the CA certificate in PEM format and store in an empty directory
2. Generate proper hash-named symbolic link for the certificate file(s) either doing
{% highlight bash %}
$ ln -s a_ca_cert.crt `openssl x509 -hash -noout -in a_ca_cert.crt`.0
{% endhighlight  %}
or using the `c_rehash` tool (available with OpenSSL, at least on Debian...)
specifying the directory where the tools should look for the cert. files and
generate the hash-named sym-links.
3. Copy the certificate file(s) and symbolic links to `/etc/ssl/certs/`
(and set properly permissions like `chown 0.0` and `chmod 644`)
4. (optional) If you use some program in Java / Mono / ... you may want also
to update their keystores. If you are using a Debian(like?) Linux distribution
it might be sufficient to execute: `$ update-ca-certificates` to do this.

For the curl you can use the -v option in case on any problem. You can also
`strace` it to see the name of the symlink it is reading (or trying to read)
while verifying the certificate of a service.


### Useful links
1. [Certificate Installation with OpenSSL - Other People's Certificates][cert_inst.1]

[cert_inst.1]:   http://gagravarr.org/writing/openssl-certs/others.shtml
