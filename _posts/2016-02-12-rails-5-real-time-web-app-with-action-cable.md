---
layout: post
title: Rails 5 - Real time Web app with action cable
comments: true
excerpt_separator: <!-- more -->
---

Rails 5 is about to be really soon and one of the very exciting features that I really would like to make use of is __Action Cable__. This feature will enable our Rails app to be able to perform push request the first time instead of relying on a 3rd party gem / plugin like Faye.

In this article, let's try to apply Action Cable feature into a traditional Todo list app and see how it will works and how it can make the normal todo list app more lively.

<!-- more -->

## 1. Install Rails 5 Beta and generate a Rails 5 app.

At the point of this article, Rails 5 is still in beta. We would need to install Rails 5 ``pre``. Here is the command to run:

{% highlight bash %}
gem install rails --pre
{% endhighlight %}

Then, let's generate a brand new Rails 5 project:

{% highlight ruby %}
rails new todolist
{% endhighlight %}

To keep it simple, we would stick with sqlite database (default) for now. However, you can change to mysql database by simply adding in ``--database=mysql`` at the end of the above ``rails new ...`` command.

## 2. Generate Item model and migrate

Now, let's churn out ``Item`` model with ``description`` text attribute and boolean attribute ``is_done``. ``is_done`` attribute is used to identify if and item has been marked as done:

{% highlight bash %}
rails g model Item description:text is_done:boolean
{% endhighlight %}

Remeber to set a default ``false`` value for ``is_done`` in the generated migration file. After that, we are good to run the migration:

{% highlight ruby %}
bundle exec rails db:migrate
{% endhighlight %}

__Note__: In Rails 5, we can now run the ``db:migrate`` using ``rails`` command instead of ``rake``. This change is for good purpose, as for new developers, they would not be confused between when to use ``rake`` and when to use ``rails``.

## 3. Generate a controller for items

Next stuff, we would need to generate a rails controller named ``ItemsController`` with index action to show the list of done and undone items:

{% highlight bash %}
rails g controller items index
{% endhighlight %}

Then add the following code for index action:

{% highlight ruby %}
# app/controllers/items_controller.rb

def index
  @done_items = Item.where(is_done: true)
  @undone_items = Item.where(is_done: false)
end
{% endhighlight %}

## 4. Add index view for items

Let's create a basic index view for listing out items. It would contain 2 lists of items: one is undone items and one is done item. On top of that, we would add in a input field for item creation:

{% highlight erb %}

<!-- app/views/items/index.html.erb -->

<input type="text" name="description"
       id="new_item" placeholder="New Item" />

<div id='to_do_list_container'>
  <h3>
    To Do List
    <span class='arrange-items' for='#to_do_list'>
      Arrange items
    <span>
  </h3>
  <ul id='to_do_list'>
    <% for item in @undone_items do %>
      <%= render 'item', item: item %>
    <% end %>
  </ul>

  <h3>
    Done List
    <span class='arrange-items' for='#done_list'>
      Arrange items
    <span>
  </h3>
  <ul id='done_list'>
    <% for item in @done_items do %>
      <%= render "item", item: item %>
    <% end %>
  </ul>
</div>
{% endhighlight %}

In addition, we would also need to add in an item unit view as following:


{% highlight erb %}
<!-- app/views/items/_item.html.erb -->

<li data-id='<%= item.id %>'>
  <input type='checkbox' id='item_<%= item.id %>'
         class='item-done' <%= item.is_done ? "checked" : "" %> />

  <span class='item-description'><%= item.description %></span>
  <span class='edit-item-description hidden'>
    <input type='text' />
  </span>
  <span class='edit-item'>edit</span>
  <span class='delete-item'>remove</span>
</li>
{% endhighlight %}

## 5. Add some basic stylesheet

You can download the stylesheet from the following URL

## 6. Generate an action cable channel

Rails 5 has added a new generator for creating channel. Let's create a channel for browser to subscribe to item creation event:

{% highlight bash %}
rails g channel item
{% endhighlight %}

then put in the following content

{% highlight ruby %}
# app/channels/item_channel.rb

class ItemChannel < ApplicationCable::Channel
  def subscribed
    stream_from "items_channel"
  end

  def unsubscribed
    # Any cleanup needed when channel is unsubscribed
  end

  def create(data)
    # data is like parameters received from front end
    Item.create({ description: data['description'] })
  end
end
{% endhighlight %}

Uncomment the action cable js code inside ``cable.coffee``

{% highlight coffee %}
# app/assets/javascripts/cable.coffee

@App ||= {}
App.cable = ActionCable.createConsumer()
{% endhighlight %}

## 7. Broadcast creation events to all subscribers

In order to broadcast a creation events to all subscribers, we would need to create a background job once an item is persited into the database. The broadcast action can run quite long, hence it's best to create a background job for it. In Rails 5, we can generate a job by running the following command:

{% highlight bash %}
rails g job ItemCreationEventBroadcastJob
{% endhighlight %}

input the following content

{% highlight ruby %}
# app/jobs/item_creation_job.rb

class ItemCreationEventBroadcastJob < ApplicationJob
  queue_as :default

  def perform(item)
    # broadcast to item_channel with the description
    # of the newly created item
    ActionCable.server.broadcast('items_channel', {
      description: render_description(item)
    })
  end

  private

  def render_description(item)
    # In Rails 5, renderer has been made public so that we can use
    # ApplicationController.renderer to render a partial
    ApplicationController.renderer.render(
      partial: 'items/item', locals: { item: item })
  end
end
{% endhighlight %}

Then inside Item model, let's create this job to perform later in the background after the item has been persisted into the database:

{% highlight ruby %}
# app/models/item.rb

class Item < ApplicationRecord
  after_create_commit do
    ItemCreationEventBroadcastJob.perform_later(self)
  end
end
{% endhighlight %}

__Note__: ``after_create_commit { ... } `` is just a short form of ``after_commit { ... }, on :create``. ``after_commit`` ensures that the entire transaction is done and committed to database before callback acts.

## 7. Edit channels/item.coffee for the received and update function

The channels javascript / coffee code plays as a bridge between the front end action with the ruby channel action.

By calling ``create`` function in ``App.item`` instance, it would dispatch the ``create`` action on ``ItemChannel`` ruby instance.

On another hand, the broadcast of the ``Item`` creation event will trigger the ``received`` function in this ``App.item`` instance.

{% highlight coffee %}
# app/assets/javascripts/channels/item.coffee

App.item = App.cable.subscriptions.create "ItemChannel",
  connected: ->
    # Called when the subscription is ready for use on the server

  disconnected: ->
    # Called when the subscription has been terminated by the server

  received: (data) ->
    # When we broadcast the item creation event,
    # all the client subscribed to item_channel
    # would have this function called, which result in calling
    # the js below
    $("#to_do_list").append(data["description"])

  create: (description) ->
    # calling perform to dispatch the create function
    # on ItemChannel with hash { description: description }
    @perform 'create', description: description
{% endhighlight %}

## 8. Edit routes.rb to enable channel

We would need to uncomment the line

``mount ActionCable.server => '/cable'``

Here is the full code for ``routes.rb``

{% highlight ruby %}
## config/routes.rb

Rails.application.routes.draw do
  # For details on the DSL available within this file, see http://guides.rubyonrails.org/routing.html

  # Serve websocket cable requests in-process
  mount ActionCable.server => '/cable'
  resources :items
end
{% endhighlight %}

## 9. Finally, add a js action for creating a new todo item on index view

{% highlight js %}
// app/assets/javascripts/application.js

$(function() {
  $("#new_item").off('keypress');
  $("#new_item").on('keypress', function(e) {
    // when user hits enter key, it would perform the creation
    if(e.keyCode == 13) {
      // App.item would dispatch the creation function
      // on ItemChannel ruby instance
      App.item.create($(this).val());
      $(this).val("");
    }
  });
});
{% endhighlight %}

When user hits the enter key (keyCode = 13) during typing inside the description box, it would trigger ``App.item.create``, which would then trigger the creation action inside ``ItemChannel`` ruby instance to create a new todo entry.


And that's it. The mini todo list should be ready to play with on [http://localhost:3000/items/index](http://localhost:3000/items/index){:target="blank"}. Here is the final demo

<p style='text-align:center;' markdown='1'><img src='/public/gifs/rails_5_action_cable.gif' alt="Final Demo" style='display:inline;'/></p>

As usual, the source code of this article is made available on this github URL - [https://github.com/jameshuynh/blog-codes/tree/master/todolist](https://github.com/jameshuynh/blog-codes/tree/master/todolist)

## 10. Now it's your turn!

You can practice action cable by continue on this mini todo list by adding edit / delete / sort action on the to do items. That way, you would understand more on how the entire action cable workflow works together.
