---
layout: post
title: Reduce database query in N+1 problem
date: 2016-03-21
comments: true
categories: [n+1 problem, rails]
tags: [n+ 1 problem, rails]
excerpt_separator: <!-- more -->
---

During development with rails, we tend to call relationship freely like ``post.comments.count``. This is usually fine. However, when we call this in a loop and go thorough an array of posts, it will cause a lot of queries into database and make the app slow down. In this article, we would go through some solutions on how to solve this problem and make our app faster.

<!-- more -->

## Problem

Let's say we are having a Rails app with the following models:

{% highlight ruby %}

# app/models/post.rb
class Post < ActiveRecord::Base
  has_many :comments, dependent: :destroy
end

# app/models/comment.rb
class Comment < ActiveRecord::Base
  belongs_to :post
end


{% endhighlight %}

a seed file

{% highlight ruby %}
# db/seeds.rb

for i in 1..10 do
  post = Post.create({ content: "Post #{i}" })
  for j in 1..10 do
    post.comments.create({ message: "Comment #{j} of Post #{i}" })
  end
end
{% endhighlight %}

Running this seeds file will create 10 posts, each post would have 10 comments

a controller and action

{% highlight ruby %}
# app/controllers/posts_controller.rb

class PostsController < ApplicationController
  def statistics
    @posts = Post.all
  end
end
{% endhighlight %}

and corresponding view

{% highlight erb %}
<!-- app/views/posts/statistics.html.erb -->

<h1>Post Statistics</h1>

<table>
  <thead>
    <tr>
      <th>Post ID</th>
      <th>Post Description</th>
      <th>Number of Comments</th>
    </tr>
  </thead>
  <tbody>
    <% @posts.each do |post| %>
      <tr>
        <td><%= post.id %></td>
        <td><%= post.content %></td>
        <td><%= post.comments.count %></td>
      </tr>
    <% end %>
  </tbody>
</table>
{% endhighlight %}

Now if you run rails server and visit this page: http://localhost:3000/posts/statistics. You will see this page:

<p style='text-align:center;' markdown='1'><img src='/public/images/n1problem.png' alt="n+1 problem" style='display:inline;'/></p>

It looks perfectly normal. However, if you check rails log, you will see something like the following

{% highlight bash %}
Started GET "/posts/statistics" for ::1 at 2016-03-23 13:28:50 +0800
Processing by PostsController#statistics as HTML
  Post Load (0.6ms)  SELECT "posts".* FROM "posts"
   (1.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 1]]
   (0.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 2]]
   (0.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 3]]
   (0.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 4]]
   (0.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 5]]
   (0.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 6]]
   (0.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 7]]
   (0.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 8]]
   (0.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 9]]
   (0.1ms)  SELECT COUNT(*) FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 10]]
  Rendered posts/statistics.html.erb within layouts/application (34.6ms)
Completed 200 OK in 805ms (Views: 799.1ms | ActiveRecord: 3.3ms)
{% endhighlight %}

This looks pretty scary, as there are totally 11 queries that hit the database. One query is to query all the posts, the rest of 10 queries are used to query the number of comments for each posts that returned from the first query. Imagine, if the first query return __N__ posts, subsequently, there would be __N__ queries to query out the number of comments. Hence, there would be __N+1__ queries that are used to query out the data for this page. This is pretty expensive as imagine there could be thousands of posts queried and the page load time would be unexpectedly long. Another issue is that we could not predict how long this could be.

## Solution

### Easy and Generic Solution

To solve the above problem with a simple solution, we could use Rails eager associations feature provided by Rails as documented [here](http://guides.rubyonrails.org/active_record_querying.html#eager-loading-associations). To do that, we can change the ``Post`` model to eager load the comment by changing the controller code like following

{% highlight ruby %}
class PostsController < ApplicationController
  def statistics
    @posts = Post.includes(:comments)
  end
end
{% endhighlight %}

and change the view code to use __.length__ instead of __.count__ like following

{% highlight erb %}
<td><%= post.comments.length %></td>
{% endhighlight %}

If you refresh the page, the page should still look the same. However, if you check the log, it should be like this by now

{% highlight bash %}
Started GET "/posts/statistics" for ::1 at 2016-03-23 13:46:03 +0800
Processing by PostsController#statistics as HTML
  Post Load (0.1ms)  SELECT "posts".* FROM "posts"
  Comment Load (0.3ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
  Rendered posts/statistics.html.erb within layouts/application (6.2ms)
Completed 200 OK in 22ms (Views: 19.6ms | ActiveRecord: 0.4ms)
{% endhighlight %}

The number of queries has been cut from __11__ queries to only __2__ queries by now. In the second query:

{% highlight sql %}
SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
{% endhighlight %}

It indeed try to query out all the comments whose ``post_id`` is inside the list id found from the post query. In this case, it is 1 to 10. Those queried out comments would be stored inside the memory and associate with corresponding post. As you can see, this number of queries can be predicted. It would be always __2__ queries regardless of how many posts that you have in your database.

Although, this solution is much faster and more predictable than original problem, the draw back of this solution is that, when you trigger an eager load an association, it means you would need to prepare a part of memory to host all the  comment objects queries out from the earger load. This also mean you are trading memory space for query time.

This would be perfectly fine if the number of objects of the association is not so many. However, it would be another problem if there are __millions__ of records in the association. For instance, some post can have millions of comments. Extract out all the comments of that particular post and store inside memory is not a wise choice. However, as I said, if you know what you are doing and you know how much data that would be loaded into the memory and you are sure you can host them inside memory, it's fine, you can use this approach.

### Better Solution but not generic

Instead of trying to eager load comments association, we could do a little better by trying to query out the number of comments right inside the query

{% highlight ruby %}
class PostsController < ApplicationController
  def statistics
    @posts = Post.joins(:comments).select("posts.*, count(comments.id) comments_count").group('posts.id')
  end
end
{% endhighlight %}

and change the view code to use __.length__ instead of __.count__ like following

{% highlight erb %}
<td><%= post.comments_count %></td>
{% endhighlight %}

Once you are done with this, refresh the page. You should see the same result. Now, let's see the log:

{% highlight bash %}
Started GET "/posts/statistics" for ::1 at 2016-03-23 14:00:37 +0800
Processing by PostsController#statistics as HTML
  Post Load (0.2ms)  SELECT posts.*, count(comments.id) comments_count FROM "posts" INNER JOIN "comments" ON "comments"."post_id" = "posts"."id" GROUP BY posts.id
  Rendered posts/statistics.html.erb within layouts/application (7.9ms)
Completed 200 OK in 29ms (Views: 24.3ms | ActiveRecord: 0.7ms)
{% endhighlight %}

As you can see, it requires only 1 query. We have joined posts table with comments table and count the comments grouped by ``post_id``. The good thing about this solution is that, it would require only 1 query, not consuming a lot of memory spaces like the earlier solution. However, the bad thing is that, as mentioned in the title, this is not a generic solution. For each of the problems like this, there would be a different way to solve and would require us to think in term of SQL query.

We still can do a little better by grouping the joins, select and group statements using model scope for later reuse:

{% highlight ruby %}
class Post < ActiveRecord::Base
  has_many :comments, dependent: :destroy
  scope :with_comments_count, -> { joins(:comments).select("posts.*, count(comments.id) comments_count").group('posts.id') }
end
{% endhighlight %}

and use it inside the controller

{% highlight ruby %}
class PostsController < ApplicationController
  def statistics
    @posts = Post.with_comments_count
  end
end
{% endhighlight %}

## Conclusion

In this article, we have gone through a sample N+1 problem and 2 solutions on how to rectify the problem:

- Easy Solution: This is a generic solution using Rails earger association loading. However, this solution is known to memory space consuming. It would be only suitable with cases whereby the number of associated objects are not so many.
- Better Solution: This is better but NOT a generic solution. For different cases, developer like us would have to think what could be done with SQL to query out the data. It would not be trivial and require some effort.

As usual, I have uploaded the source code of this article in the following URL. You are free to download and try it out and experience yourself. Happy coding!

[https://github.com/jameshuynh/blog-codes/tree/master/n1problem](https://github.com/jameshuynh/blog-codes/tree/master/n1problem)
