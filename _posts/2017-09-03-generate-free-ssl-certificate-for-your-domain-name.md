---
layout: post
title: Generate Free SSL Certificate for your domain name
date: 2017-09-03
comments: true
categories: [ssl, server, linux]
tags: [ssl, server, linut]
excerpt_separator: <!-- more -->
---

I have been purchasing SSL for my client applications using various service provider like Trustico, RapidSSL. However, the process of generating the SSL is quite a lengthy, time consuming and troublesome process. Recently, I found out that, we can actually generate SSL from a service provider - Let's Encrypt, a service that lets you generate free SSL certificate for free without a lot of hassle like a common SSL service provider. In this article, I am going to go through how to use this to generate an SSL certificate for one of my domain name.

<!-- more -->

The steps that I usually perform when I need to generate an SSL cert is go to the service provider site, fill in a lengthy form with a bunch of information about my client, company name, administrator details. And finally, the most annoying thing is fill in the approver email which ties to the domain name of the domain that I want to have the SSL covered. After that, I still need to go to the approver email inbox and click on the approve button or ask the person who owns the email to do that on my behalf. The problem is that for most of my domain names, there are no emails that are tied to that domain names. Hence, I ends up having to sign up a free Gmail or Zoho account and do all the DNS mappings (MX, DKIM, SPF if you know what I mean) before I can get access to that email. This is really a long and extremely time consuming process. I always tell myself that I need a better and faster way to generate SSL.

Luckily I found Let's Encrypt, a service to let me generate SSL certificate in a matter of minutes. Here, I am going to share how. However, you would need the following fullfil:

- The domain name that you want to have that SSL covered must be pointing to the server that you are about to run the Let's Encrypt command. Just make sure the domain name has an A record that points to the server's IP. To check if you point correctly, just do a simple ping:
- You need to make sure that both HTTP and HTTPS request can hit the server. (e.g. for EC2, you need to make sure, the Inbound in Security Group is created for both HTTP and HTTPS)

These are indeed a lot simpler and faster to have than the original way to generate the SSL certificate.

```bash
ping your-domain-name.com
PING your-domain-name (xxx.xxx.xxx.xxx): 56 data bytes
PING your-domain-name (xxx.xxx.xxx.xxx): 56 data bytes
PING your-domain-name (xxx.xxx.xxx.xxx): 56 data bytes
```

If the IP displayed in ``xxx.xxx.xxx.xxx`` is the same as your server IP, you are good to go.

Next, you would need to install ``certbot`` by issuing the following commands on your server

```bash
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx 
```

After finishing running the above commands, you can start generating the cert by issuing the following command:

```bash
sudo certbot --nginx
```

Then it will bring you through to answer a few simple questions before giving you the free SSL. There are a few options that you can do like the following to auto create the certs for apache server.

```bash
sudo certbot --apache
```

You also can just generate the SSL certificate by add in the flag:

```bash
sudo certbot --nginx certonly
```

The certificates will be created in the folder ``/etc/letsencrypt/archive/your-domain-name.com``:

```bash
root@ip-xxx.xxx.xxx.xxx:/etc/letsencrypt/archive/your-domain-name.com# ls -al
total 24
drwxr-xr-x 2 root root 4096 Sep  3 15:01 .
drwx------ 3 root root 4096 Sep  3 15:01 ..
-rw-r--r-- 1 root root 1801 Sep  3 15:01 cert1.pem
-rw-r--r-- 1 root root 1647 Sep  3 15:01 chain1.pem
-rw-r--r-- 1 root root 3448 Sep  3 15:01 fullchain1.pem
-rw-r--r-- 1 root root 1708 Sep  3 15:01 privkey1.pem
```

You can take the content of these files and create the certificate in AWS or similar services.

There are some drawbacks though:

- First, Let's Encrypt does not support wildcard SSL. Hence, if you need one, you would still need to get from a common SSL service provider. However, Let's Encrypt promises to bring this in January 2018. So stay tuned.
- Let's Encrypt gives the duration of SSL on its merit. Hence, sometimes it does not give until 1 year SSL. For some of my cases, it can only last up to 3 months and I need to renew after that. Since, it is free, there is nothing I can complain about.
- It will block the server's IP address temporarily if you keep having an error during the certificate generation.

That's all for this article that I would like to share. I hope you will enjoy the service just like what I did and start using Let's Encrypt now like me.
