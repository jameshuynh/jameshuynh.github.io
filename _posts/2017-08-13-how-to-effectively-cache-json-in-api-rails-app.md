---
layout: post
title: How to effectively cache json in API Rails app.
date: 2017-08-13
comments: true
categories: [cache, json, rails]
tags: [cache, json, rails]
excerpt_separator: <!-- more -->
---

It is quite a trend now that we use Rails API combining with a front end framework like ReactJS, AngularJS or VueJS. This is also a good way for us to streamline the data serving. All data flowing out from Rails app will just be JSON. This sparks my huge interest of finding a way to cache the returned json so that it can return faster without hitting the database. In this article, I would like to share a way on how to do effective cache json.

<!-- more -->

### 1. Build a sample app without any cache mechanism

Let's go through this by having a sample app. In this sample app, I have 2 models ``Post``:

```ruby
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
end
```

and ``Comment``

```ruby
class Comment < ApplicationRecord
  belongs_to :post
end
```

and ``Post`` has many ``Comment``

Next, let's say we have a controller and action index whereby we will need to list down all our posts and comments like below:

```ruby
class PostsController < ApplicationController
  def index
    render json: Post.includes(:comments).to_json(include: :comments)
  end
end
```

and of course a route ``posts#index``:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get 'posts/index'
end
```

This is just the worst thing that can happen that we will need to pull out everything like this. So I am just assuming that we hit the worst here and let's see how slow it can go.

Next, assuming that we have a lot of data. In this case, I am using ``seeds.rb`` to simulate the fake data:

```ruby
# db/seeds.rb
post_description =
  %(Simply dummy text of the printing and typesetting industry)

(1..10).to_a.each do |index|
  post = Post.create(title: "Post #{index}", content: post_description)
  (1..1000).to_a.each do |comment_index|
    post.comments.create(content: "Comment #{comment_index}")
  end
end
```

As you can see, we have generated 10 posts, and each post has 1000 comments. This data is quite big enough to illustrate how bad it can go when we pull out all the data and how cache can later help to speed up signifficantly the load.

Now, let's me try to hit the action index inside ``PostsController`` in localhost by trigger the ``curl` command:

```bash
time curl -s http://localhost:3000/posts/index > /dev/null
```

The output is:

```bash
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 0% cpu 1.364 total
```

and inside Rails log, for this particular request, the output is:

```bash
Started GET "/posts/index" for 127.0.0.1 at 2017-08-13 16:35:38 +0800
Processing by PostsController#index as */*
  Post Load (0.3ms)  SELECT `posts`.* FROM `posts`
  Comment Load (11.3ms)  SELECT `comments`.* FROM `comments` WHERE `comments`.`post_id` IN ('1', '2', '3', '4', '5', '6', '7', '8', '9', '10')
Completed 200 OK in 1351ms (Views: 0.2ms | ActiveRecord: 11.6ms)
```

As you can see, it takes roughly around 1.3 seconds to return back the data. This is quite a lot of time if we spend 1.3 seconds for 1 request. This means that our throughput is only:

```
1 request / 1.3s = 0.769 requests / second (req/s)
```

``0.769 req/s`` is very very low throughput even for a normal app. Imagine we have 50 users who hit this controller action, eventually the last person that will have the json return will need to wait:

```
50 requests / (0.769 req/s) = 65 seconds ~= 1 minute
```

``1 minute`` to wait for the data to come back is not something that everyone wants. Very likely, the person will just navigate away the site before the data got returned. We will need to bring up the throughput by adding caching mechanism.

### 2. Add Rails cache with Redis at the development environment

For the scope of this article, I will walk through how we are going to add cache into our development environment to evaluate the performance. It will be pretty similar to do for staging or production environment once you know how to do it in development environment.

Let's start by looking at the caching code that Rails already have in ``config/development.rb``:

```ruby
# config/development.rb

# ...
if Rails.root.join('tmp/caching-dev.txt').exist?
  config.action_controller.perform_caching = true

  config.cache_store = :memory_store
  config.public_file_server.headers = {
    'Cache-Control' => "public, max-age=#{2.days.seconds.to_i}"
  }
else
  config.action_controller.perform_caching = false

  config.cache_store = :null_store
end
# ...
```

As you can see, by default the ``cache_store`` in Rails is ``null_store``, which means no cache. This is understandable as Rails team wants to provide a development environment so that everytime we refresh, we can hit with the updated code.

To test the cache, we can simply create an empty file ``tmp/caching-dev.txt`` so that it will enter the ``if`` condition.


```bash
touch tmp/caching-dev.txt
```

Note that. the default cache store is ``memory_store``, which means it will use the machine RAM to cache the data. With ``memory_store``, we will not be able to scale the cache store to multiple machines, i.e. if you have multiple servers running the app, the cache store will not be able to share between these servers. We should use ``memcache`` or ``redis_store`` in order for the running servers to share the same cache. Let's change the ``cache_store`` to redis store:

```ruby
config.cache_store = :redis_store
```

We woudld also need ``redis_rails`` gem to be in ``Gemfile`` and have ``bundle install`` run:

```
# Gemfile
# ...
gem 'redis-rails'
# ...
```

Once you do this, you'll need to restart the development server so that it will update with the new configuration. Once the server is restarted and you try to hit the same Rails controller action, it will still be the same speed with what we have tried earlier on. To make the cache happen, what we will need to do is to change the action code to be like following:


```ruby
class PostsController < ApplicationController
  def index
    json = Rails.cache.fetch('posts') do
      Post.includes(:comments).to_json(include: :comments)
    end

    render json: json
  end
end
```

What I did was that we now introduce the block:

```ruby
Rails.cache.fetch('posts') do
  # ...
end
```

So that it will cache whatever result created inside the body of the block and cache it into our cache store with the key ``posts``. We would also need to have ``redis-server`` run:

```bash
redis-server
```

Now, let's try to hit the same controller & action again:

```bash
time curl -s http://localhost:3000/posts/index > /dev/null
```

and the output is:

```bash
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 0% cpu 1.432 total
```

This is quite slow ``1.432 seconds``. However, in cache world, they said that the warm up state is the slowest. We can revisit this again to help this later on. Now, it is a lot faster when we hit the same thing again:

```bash
time curl -s http://localhost:3000/posts/index > /dev/null
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 34% cpu 0.016 total

time curl -s http://localhost:3000/posts/index > /dev/null
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 35% cpu 0.013 total
```

This is pretty amazing as we have brought up the throughput from ``0.769 req/s`` to:

```bash
1 request / 0.013s = 76.9 requests / second (req/s)
```

In other word, the throughput has been brought up by __100 times__ by just simply introducing few lines of code.

This is awesome. However, we will need to find a way to invalidate our cache as well when our database data gets changed. At the moment, our cache key is fixed as ``posts`` string so the cache does not get updated even when we change some data in ``Post`` or ``Comment`` table.

So 2 problems that need to be solved:

- Refresh cache when data gets changed in ``Post`` or ``Comment`` table
- Warm up the cache even before the user hits the controller & action

## Problem 1: Refresh cache when data gets changed

The usual way in computer science is to invalidate the cache. However, in this case, we will do a bit differently by introducing the cache key that will get updated when data change. The cache key will be like the following inside ``Post`` model:


```ruby
class Post < ApplicationRecord
  # ...
  def self.cache_key(posts)
    {
      serializer: 'posts',
      stat_record: posts.maximum(:updated_at)
    }
  end
  #...
end
```

By using this cache key, whenever a post get updated, the result of ``posts.maximum(:updated_at)`` will be changed hence forming a new key. This has one pitfall though. When a comment beloging ti a post get updated or get added, we would also want the ``updated_at`` of the corresponding post got updated as well. To achieve that, we can simply drop in the ``touch: true`` in the relationship like below:

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true
end
```

So now we have the cache key and the correct mechanism to update the post's updated_at attribute, let's apply this into our original rails action:

```ruby
class PostsController < ApplicationController
  def index
    posts = Post.includes(:comments)
    json = Rails.cache.fetch(Post.cache_key(posts)) do
      posts.to_json(include: :comments)
    end

    render json: json
  end
end
```

If now we use curl to hit the controller & action again we can expect the same result as what we get earlier when we first put in the cache:

```bash
time curl -s http://localhost:3000/posts/index > /dev/null
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 0% cpu 1.432 total

time curl -s http://localhost:3000/posts/index > /dev/null
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 19% cpu 0.028 total
```

However, it is different this time round when we try to update a post or a comment, the cache key will be reformed. We can try that out by touching a comment from Rails console:

```ruby
irb(main):001:0> Comment.last.touch
Comment Load (0.1ms)  SELECT  `comments`.* FROM `comments` ORDER BY `comments`.`id` DESC LIMIT 1
(0.1ms)  BEGIN
SQL (0.2ms)  UPDATE `comments` SET `comments`.`updated_at` = '2017-08-13 09:31:36' WHERE `comments`.`id` = 10000
Post Load (0.1ms)  SELECT  `posts`.* FROM `posts` WHERE `posts`.`id` = 10 LIMIT 1
SQL (0.1ms)  UPDATE `posts` SET `posts`.`updated_at` = '2017-08-13 09:31:36' WHERE `posts`.`id` = 10
(5.5ms)  COMMIT
```

As you can see, when we touch a comment, the corresponding post's updated_at attribute will be updated as well. And now, if we hit the cache again:


```bash
time curl -s http://localhost:3000/posts/index > /dev/null
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 0% cpu 1.341 total

time curl -s http://localhost:3000/posts/index > /dev/null
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 19% cpu 0.025 total
```

It will be slow as first due to the new cache key but it will be fast subsequently.

We have managed to solve the first issue with refreshing the cache.

## Problem 2: Warm up the cache before the user hits the controller & action

To achieve this, we can use Rails callback ``after_save`` inside ``Post`` model to warm up the cache everytime a post entity got touched:

```ruby
class Post < ApplicationRecord
  # ...
  after_save :create_json_cache

  private

  def create_json_cache
    posts = Post.includes(:comments)
    Rails.cache.fetch(Post.cache_key(posts)) do
      posts.to_json(include: :comments)
    end
  end
end
```

With this in place, we can try to touch a comment inside rails console and see how it works:

```ruby
Comment.last.touch
Comment Load (0.3ms)  SELECT  `comments`.* FROM `comments` ORDER BY `comments`.`id` DESC LIMIT 1
  (0.1ms)  BEGIN
  SQL (0.2ms)  UPDATE `comments` SET `comments`.`updated_at` = '2017-08-13 10:07:26' WHERE `comments`.`id` = 10001
  Post Load (0.2ms)  SELECT  `posts`.* FROM `posts` WHERE `posts`.`id` = 10 LIMIT 1
  SQL (0.2ms)  UPDATE `posts` SET `posts`.`updated_at` = '2017-08-13 10:07:26' WHERE `posts`.`id` = 10
   (8.3ms)  SELECT MAX(`posts`.`updated_at`) FROM `posts` LEFT OUTER JOIN `comments` ON `comments`.`post_id` = `posts`.`id`
  Post Load (0.1ms)  SELECT `posts`.* FROM `posts`
  Comment Load (12.0ms)  SELECT `comments`.* FROM `comments` WHERE `comments`.`post_id` IN ('1', '2', '3', '4', '5', '6', '7', '8', '9', '10')
   (5.8ms)  COMMIT
=> true
```

As you can see, the cache is generated every time something got changed inside a post model. Now, let's try to hit the controller & action again:

```bash
time curl -s http://localhost:3000/posts/index > /dev/null
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 18% cpu 0.026 total
```

Now even we hit the first time after the comment get updated, the serving of json is still amazingly fast - ``0.026 seconds`` - the same speed with having the cache being served. However, we now have a problem with everytime a post or comment get updated, it will be pretty slow waiting for the cache to be generated. In order to counter this, we can offload the cache generation into a brackground process like sidekiq by using Rails Active Job.

To begin, let's create a job for this task:

```bash
rails g job create_posts_json_cache
Running via Spring preloader in process 15646
      invoke  test_unit
      create    test/jobs/create_posts_json_cache_job_test.rb
      create  app/jobs/create_posts_json_cache_job.rb
```

Then inside ``app/jobs/create_posts_json_cache_job.rb``, let's add in the code to generate out the cache:


```ruby
class CreatePostsJsonCacheJob < ApplicationJob
  queue_as :default

  def perform(*_args)
    posts = Post.includes(:comments)
    Rails.cache.fetch(Post.cache_key(posts)) do
      posts.to_json(include: :comments)
    end
  end
end
```

And then use it to replace the original cache generation code in ``Post`` model:

```ruby
class Post < ApplicationRecord
  # ...
  after_save :create_json_cache

  private

  def create_json_cache
    CreatePostsJsonCacheJob.perform_later
  end
end
```

To use the active job in combining with ``sidekiq``, we would need to add ``sidekiq`` gem to out Gemfile and have it ``bundle install``:

```ruby
# ...
gem 'sidekiq'
# ...
```

and change the active job's queue adapter to use ``sidekiq`` inside ``config/application.rb``:

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.active_job.queue_adapter = :sidekiq
  end
end
```

Next, start sidekiq using the command:

```bash
bundle exec sidekiq
```

So now everytime we run the comment update we will see that the creating cache job is sent to sidekiq instead:


```ruby
irb(main):002:0> Comment.last.touch
...
Enqueued CreatePostsJsonCacheJob (Job ID: 551d6b99-c180-4f8f-99b1-1791bee344c7) to Sidekiq(default)
   (6.1ms)  COMMIT
=> true
```

And it is extremely fast now. The creation cache is now taken care of by our sidekiq server. 

If you switch the sidekiq log, you can see the job is being executed:

```bash
2017-08-13T10:22:42.894Z 17326 TID-oupesqxl4 CreatePostsJsonCacheJob JID-04b2ee9e6280405f1cad8b3d INFO: done: 1.291 sec
```

Trying out the curl command again should still give us the fast result:

```bash
time curl -s http://localhost:3000/posts/index > /dev/null
curl -s http://localhost:3000/posts/index > /dev/null  0.00s user 0.00s system 13% cpu 0.039 total
```

That's it. We now have a closed loop of how we handle cache, invalidation and everything is fast. I hope you enjoy the article. The source code for this app is made available in the following URL so that you can check it out.

[https://github.com/jameshuynh/blog-codes/tree/master/rails-cache-demo-app](https://github.com/jameshuynh/blog-codes/tree/master/rails-cache-demo-app)
