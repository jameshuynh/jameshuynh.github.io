---
layout: post
title: Ajax Pagination on Rails and Kaminari
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


Populate some seeds data in ``seeds.rb``

{% highlight ruby %}
[
  { title: 'Seven Mobile Apps in Seven Weeks', cover: 'https://imagery.pragprog.com/products/445/7apps_xlargebeta.jpg?1453859078' },
  { title: 'Serverless Single Page Apps', cover: 'https://imagery.pragprog.com/products/453/brapps_xlargebeta.jpg?1446566716' },
  { title: 'Programming Elixir 1.2', cover: 'https://imagery.pragprog.com/products/491/elixir12_xlargebeta.jpg?1451929828' },
  { title: 'Developing for Apple Watch, Second Edition', cover: 'https://imagery.pragprog.com/products/465/jkwatch2_xlargebeta.jpg?1449158389' },
  { title: 'Rails, Angular, Postgres, and Bootstrap', cover: 'https://imagery.pragprog.com/products/448/dcbang_xlargecover.jpg?1437680108' },
  { title: 'Secure Your Node.js Web Application', cover: 'https://imagery.pragprog.com/products/443/kdnodesec_xlargecover.jpg?1433877235' },
  { title: 'Programming Phoenix', cover: 'https://imagery.pragprog.com/products/452/phoenix_xlargebeta.jpg?1441916658' },
  { title: 'Reactive Programming with RxJS', cover: 'https://imagery.pragprog.com/products/423/smreactjs_xlargecover.jpg?1438799363' },
  { title: 'Ruby Performance Optimization', cover: 'https://imagery.pragprog.com/products/425/adrpo_xlargecover.jpg?1427141274' },
  { title: 'Creating Great Teams', cover: 'https://imagery.pragprog.com/products/463/mmteams_xlargecover.jpg?1438711295' },
  { title: 'Practical Vim, Second Edition', cover: 'https://imagery.pragprog.com/products/462/dnvim2_xlargecover.jpg?1440682071' },
  { title: 'Modern Perl, Fourth Edition', cover: 'https://imagery.pragprog.com/products/458/swperl_xlargecover.jpg?1434051662' },
  { title: 'Deliver Audacious Web Apps with Ember 2', cover: 'https://imagery.pragprog.com/products/427/mwjsember_xlargecover.jpg?1433347051' },
  { title: 'Text Processing with Ruby', cover: 'https://imagery.pragprog.com/products/437/rmtpruby_xlargecover.jpg?1426186414' },
  { title: 'Pragmatic Scala', cover: 'https://imagery.pragprog.com/products/399/vsscala2_xlargecover.jpg?1442946461' },
  { title: 'Learn Game Programming with Ruby', cover: 'https://imagery.pragprog.com/products/419/msgpkids_xlargecover.jpg?1440431060' },
  { title: 'Exercises for Programmers', cover: 'https://imagery.pragprog.com/products/461/bhwb_xlargecover.jpg?1436545859' },
  { title: 'Customer Requirements', cover: 'https://imagery.pragprog.com/products/470/d-mbcreq_xlargecover.jpg?1445450768' }
].each do |book_data|
  book = Book.new({ title: book_data[:title], cover: URI.parse(book_data[:cover]) })
  book.save
end
{% endhighlight %}

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
  @books = Book.page(params[:page] || 1).per(5)
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

Add a bit of style inside ``application.css``

{% highlight css %}
.clearfix:after {
     visibility: hidden;
     display: block;
     font-size: 0;
     content: " ";
     clear: both;
     height: 0;
     }
.clearfix { display: inline-block; }
/* start commented backslash hack \*/
* html .clearfix { height: 1%; }
.clearfix { display: block; }

body {
  font-family: "Hevetica", Arial;
  font-size: 13px;
}

ul.books {
  list-style-type: none;
}

ul.books li {
  float: left;
  width: 300px;
  text-align: center;
  margin-top: 20px;
  color: #333;
  font-size: 13px;
}

ul.books li img {
  box-shadow: 0 0 5px #888;
  margin-bottom: 10px;
}

nav.pagination {
  text-align: center;
  margin-top: 50px;
  display: block;
}

nav.pagination a {
  text-decoration: none;
  border: 1px solid #eee;
  color: #0d75aa;
  padding: 10px;
  border-radius: 5px;
}

div.loading {
  text-align: center;
  margin-top: 20px;
  font-size: 13px;
  color: #333;
}
{% endhighlight %}

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
    @books = Book.page(params[:page] || 1).per(5)
    # ajax request will result in request.xhr? to be true
    # layout will be true if request is not an ajax request
    render action: :index, layout: request.xhr? == false
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


## 5. Conclusion

We have gone through how to make an ajax pagination using standard pagination gem like Kaminari. It would be beneficial if you could extend the current implementation to cater for search and filter. I hope you find this article useful and please input your comments if you have any suggestions to make this even better :-).

Last but not least, the source code of this article is made available on this github URL - [https://github.com/jameshuynh/blog-codes/tree/master/ajax-pagination-demo](https://github.com/jameshuynh/blog-codes/tree/master/ajax-pagination-demo)
