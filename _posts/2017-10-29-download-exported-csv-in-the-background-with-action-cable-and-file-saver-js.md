---
layout: post
title: Download Exported CSV in the background with Action Cable and FileSaver.js
date: 2017-10-27
comments: true
categories: [rails, action cable, file-saver]
tags: [rails, action cable, file-saver]
excerpt_separator: <!-- more -->
---

I set a rule in our company that we cannot have a long running process running in Rails controller as it will easily prolong the occupation of one of the puma/unicorn server instances. We usually have a limited number of puma/unicorn server instances and it is a bad idea of having one of them occupied for more than 1 second, as the more and the long time the puma/unicorn server instance is occupied, the less throughput we would have. This could result in slow site and bad user's experience.

One of the common tasks that can occupy the puma/unicorn server instance for a very long time is generating a CSV report task. The usual process is that a user would click on a generate CSV file button and wait there until the file is fully generated on the server and then trigger a download through Rails ``send_data`` command in a controller. In this article, I am going through on how we can solve this problem by making the report generation happen in the background and only trigger the download once it is finished generating on the server.

<!-- more -->

## 1. Create a new Rails app

From your terminal, issue the following command to create a new Rails app:

```bash
rails new rails-background-download
```

## 2. The naive way of exporting a CSV report from server

Let's generate a books controller with 2 ``download`` action and an ``index`` view:

```bash
rails g controller books
```

Inside ``books_controller``, let's add the ``download`` action like below:

```ruby
# app/controllers/books_controller.rb
class BooksController < ApplicationController
  def download
    send_data DownloadBooks.call, filename: 'books.csv'
  end
end
```

Let's create a command folder inside app folder:

```bash
mkdir app/commands
```

Then create a file ``download_books.rb`` with the following content inside ``app/commands`` folder:

```ruby
# app/commands/download_books.rb
require 'csv'
class DownloadBooks
  class << self
    def call
      sleep 10
      CSV.generate do |csv|
        (1..100).each do |_i|
          csv << %w[id title description]
        end
      end
    end
  end
end
```

I am putting ``sleep 10`` to illustrate the slowness of the process to generate out the report.

Next, create the ``index`` view for books at ``app/views/books/index.html.erb``

```erb
<div style='display: flex; justify-content: center; margin-top: 120px;'>
  <a id='download_books' class='btn btn-success' href='/books/download'>
    Download Books
  </a>
</div>
```

Lastly for this step to work, add the following to ``routes.rb``:

```ruby
Rails.application.routes.draw do
  get 'books', controller: :books, action: :index
  get 'books/download'
end
```

Now, let's turn on Rails server by issuing the following command. To clearly illustrate the bad behavior of this naive way, let's run the puma server with only 1 thread:

```ruby
puma -t 1:1
```

Then you can open 1 browsers and visit the URL:

http://localhost:3000/books/

Next onen another browser and visit http://google.com but then type the URL http://localhost:3000/books on th URL bar. 

Next, click on the Download button on the first browser and then try to visit the URL http://localhost:3000/books on the second browser and observe the effect. If you can see, although the page on http://localhost:3000/books is a very simple page, it would take at least 10 seconds to load the page. You can see the gif that I took below to see the effect.

<p style='text-align:center;' markdown='1'>
  <img src='/public/gifs/slow_loading.gif' alt="Slow Loading" style='display:inline;'/>
</p>

The reason behind this is because when we hit the Download button on the first browser, a download request is sent to our puma server. Puma server will use its single instance and process the request and hold it there until the ``download`` action finishes its running before it can process the request in the second browser. Hence it took more than 10 seconds from the loading of the page to finish loading the page on the second browser.

In production mode, you will have more than 1 puma/unicorn instances but you would also have a lot more than 2 requests like in this example. I hope you get the idea that having a puma/unicorn server instance held up to serve 1 long request is a bad thing. To make it better, we have to make the controller delegate the task to a background process and finish the request as soon as possible. It's quite simple task to do with long processing process that's not related to browser (like upload a video to external server) but it's pretty tricky when dealing with this particular case where we have to send back the exported report back to the client as a downloaded file.

## 3. Implement a way to export a CSV report from server without occupying a puma/unicorn server for long

To implement this, we would need to use Action Cable which is introduced in Rails 5. The general idea is let the controller which handle the report download request delegate the long processing task to an Active Job, which will run in the background and immediately return 'ok' to the browser. Meanwhile, the Active Job will run the long processing report generation and send back the report data through Action Cable and trigger a download on the browser.

Firstly, let's add ``jQuery`` and ``file-saver`` to Rails by issuing the yarn command:

```bash
yarn add jquery
yarn add file-saver
```

Then add in the following line to ``app/assets/javascripts/application.js``:

```js
// app/assets/javascripts/application.js
// ...
//= require jquery
//= require file-saver/FileSaver
// ...
```

I am not a fan of ``turbolinks`` so I removed the line ``//= require turbolinks`` in this ``application.js`` file but you could keep it. In this same file ``application.js`` add in the following function to help to generate a unique universal ID:

```js
// app/assets/javascripts/application.js
// ...
function generateUUID() {
  var S4 = function() {
    return (((1 + Math.random()) * 0x10000) | 0).toString(16).substring(1);
  };
  return (
    S4() +
    S4() +
    "-" +
    S4() +
    "-" +
    S4() +
    "-" +
    S4() +
    "-" +
    S4() +
    S4() +
    S4()
  );
}
```

Secondly, let's create an Action Cable channel to handle file download by issuing the following command:

```bash
rails g channel download
```

Then change the content of ``app/assets/javascripts/channels/download.js`` by the code below:

```js
function subscribeDownloadChannel(uuid, callback) {
  App.download = App.cable.subscriptions.create(
    { channel: "DownloadChannel", uuid: uuid },
    {
      connected: function() {
        callback();
      },

      disconnected: function() {},

      received: function(data) {
        var blob = new Blob([data.csv], {
          type: "text/csv;charset=utf-8"
        });

        saveAs(blob, "books.csv");

        $("#download_books")
          .html("Download Books")
          .removeAttr("disabled");

        App.download.unsubscribe();
        App.cable.disconnect();
        delete App.download;
      }
    }
  );
}
```

As per the above code, I wrapped the generated code inside a function called ``subscribeDownloadChannel`` with ``uuid`` and ``callback`` as the parameters. The ``uuid`` will be generated everytime the download button is clicked and pass to this function to start a subscription to ``DownloadChannel`` with a univeral unique id. 

Once the subscription received a data from Action Cable server (as shown in ``received`` function), it will create a ``blob`` and trigger a Save As with the received content. The remaining code will clean up the channel, remove the subscription and stop the Action Cable.

Thirdly, modify the file `app/channels/download_channel.rb`` with the following content.

```ruby
# app/channels/download_channel.rb
class DownloadChannel < ApplicationCable::Channel
  def subscribed
    stream_from "downloads_channel_#{params[:uuid]}"
  end

  def unsubscribed; end
end
```

In ``subscribed`` function, we will start the streaming from ``downloads_channel_#{params[:uuid]}``. This ``params[:uuid]`` is received when you trigger the above command:

```js
App.cable.subscriptions.create(
  { channel: "DownloadChannel", uuid: uuid },
  // ...
)
```

Fourthly, let's create an Active Job called ``AvailableDownloadBroadcastJob``:

```bash
rails g job AvailableDownloadBroadcast
```

Then fill in the following code in the newly generated job file:

```ruby
# app/jobs/available_download_broadcast_job.rb
class AvailableDownloadBroadcastJob < ApplicationJob
  queue_as :default

  def perform(uuid)
    csv = DownloadBooks.call
    ActionCable.server.broadcast(
      "downloads_channel_#{uuid}", csv: csv
    )
  end
end
```

This is the mentioned delegated job that will perform the long processing process and call action cable to broadcast the event with the content of the CSV after the long processing process has been completed.

Fifthly, modify the action ``download`` in ``BooksController`` to delegate the long processing task to a ``AvailableDownloadBroadcastJob``

```ruby
# app/controllers/books_controller.rb
class BooksController < ApplicationController
  def download
    AvailableDownloadBroadcastJob.perform_later(params[:uuid])
    render json: { result: :ok }
  end
end
```

Lastly, modify the file ``app/views/books/index.html.erb`` to trigger the Action Cable subscription before requesting for the CSV report:

```erb
<!-- app/views/books/index.html.erb -->

<div style='display: flex; justify-content: center; margin-top: 120px;'>
  <a id='download_books' class='btn btn-success' href='/books/download'>Download Books</a>
</div>

<script language='javascript'>
  $(function() {
    $('#download_books').click(function(e) {
      e.preventDefault();
      var uuid = generateUUID();
      $(this).html('Downloading...').attr('disabled', 'disabled');
      var url = $(this).attr('href') + '?uuid=' + uuid;
      subscribeDownloadChannel(uuid, function() {
        $.get(url);
      });
      return false;
    })
  });
</script>
```

When hitting the download button, we will turn the button to show Downloading and disable the button, then append the UUID to the URL and then subscribe to download channel with the just generated uuid. Upon the connection is established, it will start requesting for the report in the command ``$.get(url)``. The report is generated on the background process and will eventually call back to the ``received`` callback in `app/assets/javascripts/channels/download.js`` to complete the cycle. 

Now you can try the same steps of concurrent loading 2 browsers as before and notice the difference. The second browser will be a lot more responsive. It can load the request without any issue.

<p style='text-align:center;' markdown='1'>
  <img src='/public/gifs/fast_loading.gif' alt="Fast Loading" style='display:inline;'/>
</p>

Finally, I have published the code here for your reference:

[https://github.com/jameshuynh/blog-codes/tree/master/rails-background-download](https://github.com/jameshuynh/blog-codes/tree/master/rails-background-download)
