---
title: Configuring Nginx for Let's Encrypt
date: 2017-02-25
tags:
  - web
  - security
  - perl
  - nginx
  - letsencrypt
---

Mojolicious apps can use [Mojo::ACME](https://metacpan.org/pod/Mojolicious::Plugin::ACME) to help issue a certificate for a single appplication. It registers routes which handle the challenges and adds commands which requests new certificates. That said, once you have more than one application and/or multiple sites in the same nginx virtual host, having that capability as a per-application plugin becomes a hinderance not a help.

In this article I will show how you can use a dummy Mojolicious app to issue certs for all of your virtual hosts.

## The Dummy Application

At least until I modify Mojo::ACME to provide a built-in dummy application server, you still need a very simple application which only loads the plugin.

    # mojo-acme.pl

    use Mojolicious::Lite;

    plugin 'ACME';
    
    app->start;


This application only needs to be running when issuing certificates. Start it on an unpriviledged port (say 8000) by running `$ perl mojo-acme.pl -l 'http://*:8000'`.

Then add a virtual-host for it in your nginx configuration. Since the other applications reference it, it needs to be loaded early, so lets call it `01-mojo-acme`.

    # /etc/nginx/sites-available/01-mojo-acme

    upstream mojo-acme {
      server 127.0.0.1:8000;
    }

## Application Virtual-Host Files

Now for your other applications, start with a skeleton like the following:


    # /etc/nginx/sites-available/myapp

    upstream myapp {
      server 127.0.0.1:8080;
    }
    server {
      listen 80;
      server_name myapp.myhost.com;
      include /etc/nginx/common/http-redirects;
    }
    server {
      listen 443 ssl;
      server_name myapp.myhost.com;
      location / {
        proxy_pass http://myapp;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
      }
      include /etc/nginx/common/ssl-settings;
    }
    
Each application should be named differently (this one is called `myapp`) and should use its own `server_name` (where this one is `myapp.myhost.com`). The bulk of this configuration is the standard [Mojolicious NGINX configuration](http://mojolicious.org/perldoc/Mojolicious/Guides/Cookbook#Nginx). There are a few major differences. The first is that it defines handlers both for port 80 (http) and port 443 (https). Each section includes a file from `/etc/nginx/common/` which is a folder you'll need to create.

In the port 80 section it includes a file which contains the following:

    # /etc/nginx/common/http-redirects

    location /.well-known/ {
      proxy_pass http://mojo-acme;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
      return 301 https://$server_name$request_uri;
    }

This file does two things. It siphons http traffic bound for certificate challenges and it directs it to the dummy app we defined earlier. Then it takes all other http traffic and responds to requests with a redirect to https. This protects users from forgetting to type `https://` before your site name, their browser will take them to the SSL protected service automatically.

The port 443 section of the virtual-host file also includes a second file which should contain something like the following:

    # /etc/nginx/common/ssl-settings

    ssl_certificate     /path/to/cert.crt;
    ssl_certificate_key /path/to/cert.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
    ssl_dhparam         /path/to/dhparams.pem;
    ssl_session_cache   shared:SSL:10m;
    ssl_prefer_server_ciphers on;
    
This file tells the application where to find your certificate and key files as well as a file defining a set of Diffie-Hellman parameters. You can generate the latter by running `$ openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048` which will take a while. The certificate and key are generated by the ACME plugin, which will be explained soon.

The rest of the parameters configure the site to use recommended modern security practices. As of the writing of this post, that configuration earns an A rating from [ssllabs.com](https://ssllabs.com) for the author's own sites.

Finally, there is no requirement that the target of the virtual-host application be a Mojolicious application. The author of this article has five virtual hosts in his current configuration. Three are Mojolicious applications, one is a PHP application, and one is a simple file-based NGINX service. The other two virtual-host configurations have difference of course, but they still include the two files and are identical in the port 80 configuration and ssl configuration.

## Issuing the Certificate

If you haven't started the dummy app as shown above, do so now. Also be sure that all of your desired vitual hosts are configured as shown above and are enabled in nginx.

NOTE! Before you continue, know that on all of these commands you can pass a `-t` flag to use a testing server provided by Let's Encrypt. In this way you can try out the service without affecting your certificate issuance rate-limit. The test servers generate valid certificates but aren't signed, so they aren't very useful other than testing.

Now if you don't already have an account key with Let's Encrypt you can create one by running `$ perl mojo-acme.pl acme account register`. This will create a file called `account.key`. Keep this file safe, you will need it for issuing certificates and all other actions you ever take for these domains.

Finally to issue the certificates, run the command `$ perl mojo-acme.pl acme cert generate -a account.key -n mydomain mydomain.com myapp.mydomain.com myotherapp.mydomain.com`. The `-a` flag tells it where to find your account key file (where `account.key` in the working directory is the default). The `-n` flag tells it what to name the generated files; which is completely arbitrary. Finally pass all the domains that you want to issue the cert for. Customarily you would pass the toplevel domain first if you are including that one.

Once the certificates are generated, move the certs to the paths you listed in `/etc/nginx/common/ssl-settings`, or else change those paths to where you want those cerificate files to remain.

Finally, once you reload your nginx configuration (when using upstart this is `$ sudo service nginx reload`), your sites should all be available using the new certificates!

In 3 months you will need to reissue the certificates, but now you only need to run that same certificate issuance command and the new files will be generated! Indeed the remaining steps are simple enough that they could be a cron job that runs periodically (say every 2 months) to keep your certificates valid indefinitely.

For future reference, the example files are collected in a gist for ease of access at [https://gist.github.com/jberger/d04cbfa20d3a269eb1f7868e2b1f432c](https://gist.github.com/jberger/d04cbfa20d3a269eb1f7868e2b1f432c).

(Originally published at <https://asyncthoughts.tumblr.com/post/157702503315/configuring-nginx-for-lets-encrypt>).
