---
layout: post
title: Hello world
excerpt_separator: <!-- more -->
---

Hello world Content

<!-- more -->

{% highlight ruby %}
def show
  @widget = Widget(params[:id])
  respond_to do |format|
    format.html # show.html.erb
    format.json { render json: @widget }
  end
end
{% endhighlight %}
