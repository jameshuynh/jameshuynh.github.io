---
layout: post
title: Fix head font cut off for iOS
date: 2016-03-04
comments: true
categories: [ios, font]
tags: [ios, font]
excerpt_separator: <!-- more -->
---

iOS development is great if we use the default font. However, it is a bit buggy when we use a custom font. In my case, I find one particular font that has the top of the font chopped off. In this article, I would like to share how I can fix that

<!-- more -->

## 1. Generate an XML configurator for the faulty font

Open your terminal and issue the following command.

{% highlight bash %}
ftxdumperfuser -t hhea -A d "Linotype - UniversLTStd-Cn.otf"
{% endhighlight %}

This would generate an XML file configurator for that font. In my case, it is ``Univers LT Std 57 Condensed.hhea.xml``.

## 2. Adjust ascender and descender inside the XML configurator

Open the generated XML file in earlier step by using your favorit editor or use vim by issuing the following command:

{% highlight bash %}
vim "Univers LT Std 57 Condensed.hhea.xml"
{% endhighlight %}

Once you open the file, look for ``<hheadTable`` tag and ``ascender`` and ``descender`` attribute. If your font has head cut off, then you should __increase ascender__ and __decrease descender__. On another hand, if your font baseline is too low from what you expect it is supposed to be, you should __decrease ascender__ and __increase descender__ instead.

Here is what look like on my XML configurator file:

{% highlight xml %}
<hheaTable
	versionMajor="1"
	versionMinor="0"
	ascender="840"
	descender="-376"
  ...
{% endhighlight %}

## 3. Generate back the font from the new XML configurator

After you have done with the editing on step 2, it's time to generate back the font by issuing the following command:

{% highlight bash %}
ftxdumperfuser -t hhea -A f "Linotype - UniversLTStd-Cn.otf"
{% endhighlight %}

The new font file would have your edited configuration that you have editted in step 2. Now you can try to run the iOS app again to verify the font has taken effect. Note that you would need to do need a few times from step 1 to 3 as the changes that you made in step 2 might not be sufficient.
