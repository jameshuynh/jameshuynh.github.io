---
layout: post
title: Ajax Pagination on Rails and Kaminari
date: 2016-01-28
comments: true
excerpt_separator: <!-- more -->
---

One of the most common tasks that our team always need to do is to perform pagination on a list of products or items. Usual pagination with Kaminari gem and page reload is good but with a bit addition on Ajax would make the application snappier and more user friendly.

In this post, I would like to introduce the way that I usually perform to convert a normal pagination to Ajax Pagination.
<!-- more -->

## 1. Create a new Rails app

First, let's create a new Rails app

{% highlight bash %}
rails new ajax-pagination-demo
{% endhighlight %}

## 2. Generate Book model and populate some demo data

First, let add Kaminari and Paperclip gem into our Gemfile
{% highlight ruby %}
gem 'kaminari', '~> 0.16.3'
gem 'paperclip', '~> 4.3.2'
{% endhighlight %}

Then generate Book Model and add cover as a paperclip attachment

{% highlight bash %}
rails g model Book title
rails g paperclip Book cover
{% endhighlight %}

And run database migration

{% highlight bash %}
bundle exec rake db:migrate
{% endhighlight %}

Add paperclip code inside ``book.rb``
{% highlight ruby %}
class Book < ActiveRecord::Base
  has_attached_file :cover, styles: { thumb: "220x310>" }
  validates_attachment_content_type :cover, content_type: /\Aimage\/.*\Z/

  before_post_process on: :create do
    if cover_content_type == 'application/octet-stream'
      mime_type = MIME::Types.type_for(cover_file_name)
      self.cover_content_type = mime_type.first.to_s if mime_type.first
    end
  end
end
{% endhighlight %}


Populate some seeds data in ``seeds.rb``, which you can copy the seeds file from here [http://bit.ly/1TsNZiT](http://bit.ly/1TsNZiT){:target="_blank"}

Finally, run the seed file to populate data

{% highlight bash %}
bundle exec rake db:seed
{% endhighlight %}

## 3. Create controller and index view

Let's generate books controller with index action:

{% highlight bash %}
rails g controller books index
{% endhighlight %}

Move to ``books_controller.rb`` and add index action code:

{% highlight ruby %}
def index
  @books = Book.page(params[:page] || 1).per(4)
end
{% endhighlight %}

Then create ``index.html.erb`` inside ``views/books``:

{% highlight erb %}
<div id='books_container'>
  <ul class='books clearfix'>
    <% @books.each do |book| %>
      <li>
        <%= image_tag book.cover.url(:thumb) %>
        <br/>
        <%= book.title %>
      </li>
    <% end %>
  </ul>

  <%= paginate @books %>
</div>
{% endhighlight %}

Add a bit of style inside ``application.css`` - which you can copy from here [http://bit.ly/1KLNpWg](http://bit.ly/1KLNpWg){:target="_blank"}


Now, if you visit [http://localhost:3000/books/index](http://localhost:3000/books/index){:target="blank"}, the normal pagination should work perfectly okay. Now we are going forwards to ajaxifying the pagination.

## 4. Convert the current pagination into Ajax Pagination

Let's open ``application.js`` and add in some Javascript delegation code.

{% highlight js %}
$(function() {
  $(document.body).off('click', 'nav.pagination a');
  $(document.body).on('click', 'nav.pagination a', function(e) {
    e.preventDefault();
    var loadingHTML = "<div class='loading'>Loading...</div>";
    $("#books_container").html(loadingHTML).load($(this).attr("href"));
    return false;
  });
});
{% endhighlight %}

What we have done in the above js code is to turn on the delegation from document body to each pagination link so that when a user clicks on a link, it will stop all the default page load and instead now display a loading message and try to load the HREF of the a tag using Ajax load instead. This way, the content would now be served via Ajax instead of usual entire page reloading.

One more thing that we would have to change on our index action is that we would have to detect the ajax load and serve the content of the action without the layout so that only the needed HTML is returned:

{% highlight ruby %}
class BooksController < ApplicationController
  def index
    @books = Book.page(params[:page] || 1).per(4)
    # ajax request will result in request.xhr? not nil
    # layout will be true if request is not an ajax request
    render action: :index, layout: request.xhr? == nil
  end
end
{% endhighlight %}

Up to this point, our Ajax pagination has been successfully implemented. However, there is one small glitch that we would tend to over look, which is the URL of the page when user clicks over a page. Currently, it would remain the same.

For instance, if you are on page 1, your URL would be ``http://localhost:3000/books/index?page=1``. Now, if a user clicks on page 2, because of the Ajax nature, the content would be changed to content of page 2, but the URL would still be ``http://localhost:3000/books/index?page=1``. In order to fix that, we would need to change the earlier js by adding the history of the page

{% highlight js %}
$(function() {
  var loadingHTML = "<div class='loading'>Loading...</div>";

  $(document.body).off('click', 'nav.pagination a');
  $(document.body).on('click', 'nav.pagination a', function(e) {
    e.preventDefault();
    var url = $(this).attr("href")
    $("#books_container").html(loadingHTML).load(url, function() {
      // push state after the content has finished loading to update the URL and save in history stack
      window.history.pushState(url, window.title, url);
    });
    return false;
  });

  $(window).bind('popstate', function(event) {
    var url = location.href;
    // reload HTML once user presses back / forward button
    $("#books_container").html(loadingHTML).load(url);
  });
});
{% endhighlight %}

And that's it. Here is the final demo

<p style='text-align:center;' markdown='1'><img src='/public/gifs/ajax-pagination-demo.gif' alt="Final Demo" style='display:inline;'/></p>

## 5. Conclusion

We have gone through how to make an ajax pagination using standard pagination gem like Kaminari. It would be beneficial if you could extend the current implementation to cater for search and filter. I hope you find this article useful and please input your comments if you have any suggestions to make this even better :-).

Last but not least, the source code of this article is made available on this github URL - [https://github.com/jameshuynh/blog-codes/tree/master/ajax-pagination-demo](https://github.com/jameshuynh/blog-codes/tree/master/ajax-pagination-demo)

## 6. There are 4 Learning Points in this article

##### 1. Paperclip photo can be extracted from a URL by passing a URI object into the paperclip attribute

{% highlight ruby %}
Book.create({ title: 'Customer Requirements', cover: URI.parse('https://imagery.pragprog.com/products/470/d-mbcreq_xlargecover.jpg?1445450768') })
{% endhighlight %}

##### 2. You can check if a request is an ajax request by using the code

{% highlight ruby %}
# request.xhr? will be true if it is an ajax rqeuest and nil otherwise
request.xhr? != nil
{% endhighlight %}

##### 3. Browser URL state can be changed by the javascript code

{% highlight javascript %}
var stateObj = { foo: "bar" };
window.history.pushState(stateObj, "Page Title", "/new-url");
{% endhighlight %}

This will cause the URL bar to display ``http://current-domain-name.com/new-url``, but won't cause the browser to load ``/new-url`` or even check that ``/new-url`` exists.

##### 4. Hooking an event into pop state event

When user clicks on Back / Forward button on the browser, you can bind a havascript function as its callback. This would help our application load approriate URL


{% highlight javascript %}
$(window).bind('popstate', function(event) {
  var url = location.href;
  // reload HTML once user presses back / forward button
  $("#books_container").html(loadingHTML).load(url);
});
{% endhighlight %}
