---
layout: post
title: SSH Remote Access a server through another server
date: 2016-02-15
comments: true
excerpt_separator: <!-- more -->
---

Recently, one of the app that I develop for our client would need to be hosted on a server that can only be accessed through another server. That has acaused me quite a few problems with ssh remote access and even deployment. In this article, I would like to share how I overcome the problem to easily remote access to the targeted server and deploy the app smoothly just like I have direct access to the targeted server.

<!-- more -->

## 1. The naive way

The naive way is to ssh remote to server A. Then once I am inside server A, I would ssh remote to server B:

{% highlight bash %}
ssh <SERVER_A_USERNAME>@<SERVER_A_IP>
## Now once we are inside SERVER_A
ssh <SERVER_B_USERNAME>@<SERVER_B_IP>
{% endhighlight %}

For instance:

{% highlight bash %}
ssh ubuntu@192.168.0.1
## Now once we are inside 192.168.0.1
ssh ubuntu@192.168.0.2
{% endhighlight %}

This method works but it's slow and not efficient. You would need to wait for the remote access done on server A before you can key in the second command to remote access to server B. In addition, if you know capistrano deployment, this way will definitely not work if you plan to deploy an app to server B.

## 2. The ssh -t way

By trigger the following command:

{% highlight bash %}
ssh -t <SERVER_A_USERNAME>@<SERVER_A_IP> \
ssh <SERVER_B_USERNAME>@<SERVER_B_IP>
{% endhighlight %}

For instance:

{% highlight bash %}
ssh -t ubuntu@192.168.0.1 ssh ubuntu@192.168.0.2
{% endhighlight %}

We would be able to ssh remote access server B with just one single command. This way would be better than the first way as you will not need to wait for the remote access to server A to be established before you can key in the next command. This will straight away fire the ssh remote access command to server B once the remote access to server A is established. However, again, this will not be helpful if you use capistrano to deployment to deploy to server B.

## 3. The best way

In order to achieve this way, you would need to ensure netcat (nc) is available on your middle man server. For instance, if you plan to deploy to server B and server B can only be accessed via server A. You would need to install netcat on server A using the following command:

{% highlight bash %}
sudo apt-get install netcat
{% endhighlight %}

The second thing that you would need to ensure is having your public key (id_rsa.pub) deposit in ``~/.ssh/authorized_keys`` on both server A and server B.

Once you have done with those 2 prerequisites, edit your ``~/.ssh/config`` and add in the following entry

{% highlight bash %}
Host <SOME_ALIAS_NAME>
  User <SERVER_B_USERNAME>
  HostName <SERVER_B_IP>
  ProxyCommand /usr/bin/ssh -p<SERVER_A_SSH_PORT> <SERVER_A_USERNAME>@<SERVER_A_IP> nc %h <SERVER_B_SSH_PORT>
{% endhighlight %}

Then you can direct access server B through the following ssh command:

{% highlight bash %}
ssh <SOME_ALIAS_NAME>
{% endhighlight %}

__NOTE__: < ... > is a variable, you would need to supply a correct value.

You will not need to support the port number and username as on the last ssh command, we already specify a port number and username on the target host in ``~/.ssh/config`` earlier.

Here is an example that I have on my ``~/.ssh/config``. Note that, I only have access to ``192.168.0.1`` and do not have direct access to ``192.168.0.2``:

{% highlight bash %}
Host internal
  User deployer
  HostName 192.168.0.2
  ProxyCommand /usr/bin/ssh -p2345 ubuntu@192.168.0.1 nc %h 2346
{% endhighlight %}

And I can run the following ssh command to login to server ``192.168.0.2``:

{% highlight bash %}
ssh internal
{% endhighlight %}

With this, you can use capistrano and specify the server as ``internal`` and username as ``deployer``. Capistrano would deploy just like you have direct access to ``192.168.0.2``.
