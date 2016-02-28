---
layout: post
title: Capistrano Deployment Behind a Proxy
date: 2016-02-28
comments: true
categories: [capistrano, proxy, deployment]
tags: [capistrano, proxy, deployment]
excerpt_separator: <!-- more -->
---

It's best if we can deploy our app into an online VPS. However, sometimes we would still need to deploy our app to a server behind a VPN. In this article, I would like to share how we can setup to ease the process of installing gems, gem bundling and git clone / push in a server behind a VPN

<!-- more -->

## Specify HTTP/HTTPS proxy

In order to use a proxy, we would need to indicate the http/https proxy by exporting ``all_proxy``, ``http_proxy`` and ``https_proxy``. To do that, let's add in the following lines into ``~/.bashrc``

{% highlight bash %}
# ~/.bashrc

export all_proxy=http://<username>:<password>@<ip_address>:<port>
export http_proxy=http://<username>:<password>@<ip_address>:<port>
export https_proxy=http://<username>:<password>@<ip_address>:<port>
{% endhighlight %}

Then reload ``~/.bashrc``:

{% highlight bash %}
. ~/.bashrc
{% endhighlight %}

## Configure wget to use HTTP/HTTPS proxy

You would very likely need wget to make use of the http/https proxy. To set that up, edit ``~/.wgetrc`` and add in the following lines

{% highlight bash %}
# ~/.wgetrc

use_proxy=yes
http_proxy=http://<ip_address>:<port>
https_proxy=http://<ip_address>:<port>
proxy_user=<proxy_username>
proxy_password=<proxy_user_password>
{% endhighlight %}

You can then try to run the following command to see if it works:

{% highlight bash %}
wget http://google.com
{% endhighlight %}

To temporarily disable a proxy, you can append ``--no-proxy`` after the above command:

{% highlight bash %}
wget http://google.com --no-proxy
{% endhighlight %}

## Configure Git to use HTTP/HTTPS Proxy

Next, we would need to setup our Git so that when we do git clone, or pull, it will go through a proxy. It's difficult to channel the git protocol through a proxy. However, it's easier to channel a git request via http/https request. Hence, what we could do is to replace the ``git@`` portion in ``git clone git@...`` with ``git clone https://...``. In order to do that, open ``~/.gitconfig`` and add the following lines:

{% highlight bash %}
# ~/.gitconfig
[credential]
	helper = cache --timeout=86400
[url "https://github.com/jameshuynh/SomeProject.git"]
	insteadOf = git@github.com:jameshuynh/SomeProject.git
{% endhighlight %}

__Note__: You would need to replace releal project path.

With this, when we run:

{% highlight bash %}
git clone git@github.com:jameshuynh/SomeProject.git
{% endhighlight %}

Git will know and convert the command to

{% highlight bash %}
git clone https://github.com/jameshuynh/SomeProject.git
{% endhighlight %}

Hence this ``git clone`` command will go through the http/https proxy. One thing to note is that the first time that you git clone / pull via http/https, git will prompt you for username & password, which you would need to key in. Subsequently, because the line ``helper = cache --timeout=86400``, git would cache your username & password. In this configuration, I am putting ``86400`` seconds, but it's up to you to adjust how long git should remember your username & password.

## Configure Gem Installation to use HTTP/HTTPS Proxy

Finally, we would need to setup proxy for our ``gem install`` or ``bundle install``. In order to do that, edit ``~/.gemrc`` and add in the following lines:

{% highlight bash %}
# ~/.gemrc
http_proxy: http://<username>:<password>@<ip_address>:<port>
https_proxy: http://<username>:<password>@<ip_address>:<port>
{% endhighlight %}

That's all. You are now ready to deploy from capistrano just like you do on a server outside VPN.
