---
layout: post
title: Save IRB and Rails Console history 
comments: true
excerpt_separator: <!-- more -->
---

This trick is super useful when you want to save the history of the commands that you have typed earlier in IRB or Rails console so that it can re-used back when you need. In order to enable this feature, follow the following steps:

<!-- more -->

## 1. Create ~/.irbrc or edit your current ~/.irbrc

{% highlight bash %}
vim ~/.irbrc
{% endhighlight %}

## 2. Add in the irb saving history extension require

Add in the following line:

{% highlight ruby %}
require 'irb/ext/save-history'
{% endhighlight %}

## 3. Optionally, you can configure the number of commands to save

On the same ``~/.irbrc`` file, add in this line:

{% highlight ruby %}
IRB.conf[:SAVE_HISTORY] = 1000
{% endhighlight %}

*Notes*: The history of your commands would be saved inside ``~/.irb_history``
