---
layout: post
title: Build Chat with React JS and Rails Action Cable
date: 2017-07-30
comments: true
categories: [rails, react js, chat]
tags: [rails, react js, chat]
excerpt_separator: <!-- more -->
---

Rails 5 comes up with Action Cable and it is a wonderful feature as usually to do a real time feature, we would need to rely on NodeJS to have implement a websocket. By combining with React JS on front end, we can easily build a chat application with Rails backend API and React JS front end to render the chat messages. In this article, let go through how to implement this solution.

<!-- more -->




## 1. Create a Rails app with only API to support real time chat

Create a new Rails app with MySQL database (or any database that you prefer) and limit this Rails app to have only API

```bash
rails new chat-app --database=mysql --api
```

Create a ``ChatMessage`` model by issuing the command:

```bash
rails g model chat_message content:text
```

then run the database creation and migration:

```bash
bundle exec rails db:create
bundle exec rails db:migrate
```

Next, let's create a chat channel by creating a file ``chat_channel.rb`` in ``app/channels``

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from 'chat_channel'
  end

  def unsubscribed; end

  def create(opts)
    ChatMessage.create(
      content: opts.fetch('content')
    )
  end
end
```

We also need to create an event broadcast job class ``chat_message_creation_event_broadcast_job.rb`` inside folder ``app/jobs/``

```ruby
# app/jobs/chat_message_creation_event_broadcast_job.rb
class ChatMessageCreationEventBroadcastJob 
< ApplicationJob
  queue_as :default

  def perform(chat_message)
    ActionCable
      .server
      .broadcast('chat_channel',
                 id: chat_message.id,
                 created_at: chat_message.created_at.strftime('%H:%M'),
                 content: chat_message.content)
  end
end
```

and then modify ``chat_message.rb`` to have the relationship with the chat_room and to send the broadcast event after it is created

```ruby
# app/models/chat_message.rb
class ChatMessage < ApplicationRecord
  after_create_commit do
    ChatMessageCreationEventBroadcastJob.perform_later(self)
  end
end
```

We also need ``rack-cors`` gem to let the request call from cross domain. Add this line into Gemfile

```ruby
# Gemfile
gem 'rack-cors'
```

and then run

```bash
bundle install
```

Then set up to allow the whilelisted domain to send request across. Add the following code to ``application.rb``:

```ruby
config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'
    resource '*', headers: :any, methods: %I[get post options]
  end
end
```

Note that, you will need to adjust the ``origins`` and ``resources`` in the above code to prevent any unexpected requests from other domains

And finally, add an Action Cable routes at ``routes.rb``

```ruby
# config/routes.rb
mount ActionCable.server => '/cable'
```

## 2. Create a React app to show the chat messages

Let install ``create-react-app`` in order to create a react app easily

```bash
npm install -g create-react-app
```

Then create a React app by issuing a command:

```bash
create-react-app chat-app-react
```

Then add action cable package:

```bash
npm i actioncable --save
```

Now, inside ``App.js``, let's start to put in some basic interface for the chat:

```js
import React, { Component } from 'react';
import logo from './logo.svg';
import './App.css';

class App extends Component {
  render() {
    return (
      <div className='App'>
        <div className='stage'>
          <h1>Chat</h1>
          <div className='chat-logs'>
          </div>
          <input
            type='text'
            placeholder='Enter your message...'
            className='chat-input'/>
          <button className='send'>
            Send
          </button>
        </div>
      </div>
    );
  }
}

export default App;
```

Next, inside ``class App`` add in a React State to keep track of the current chat message:

```js
// ...
constructor(props) {
  super(props);
  this.state = {
    currentChatMessage: ''
  };
}

updateCurrentChatMessage(event) {
  this.setState({
    currentChatMessage: event.target.value
  });
}
// ...
```

And chat the input field to use the state and the callback

```js
<input
  value={ this.state.currentChatMessage }
  onChange={ (e) => this.updateCurrentChatMessage(e) }
  type='text'
  placeholder='Enter your message...'
  className='chat-input' />
```

This ``onChange`` callback will help us keeping track of what user has typed in the chat box.

Next, prepare the Action Cable socket by creating a function as shown below:


```js
// ...
createSocket() {
  let cable = Cable.createConsumer('ws://localhost:3001/cable');
  this.chats = cable.subscriptions.create({
    channel: 'ChatChannel'
  }, {
    connected: () => {},
    received: (data) => {
      console.log(data);
    },
    create: function(chatContent) {
      this.perform('create', {
        content: chatContent
      });
    }
  });
}
// ...
```

Let's look at this function.

- First we create an action cable object pointing to websocket ``localhost:3001/cable``. This will be the Rails URL that we will run later on.
- Then we create an instance variable called ``chats`` which is an Action Cable subscription with the channel ``ChatChannel``.
- This chat subscription has 2 callbacks which are ``connected``, ``received`` and 1 action which is ``create``. Our goal is to call this ``create`` function when we try to send a chat message over and append the chat logs in the ``received`` calback function.

Now, to start using this function, call it inside ``componentWillMount``:

```js
componentWillMount() {
  this.createSocket();
}
```

Next, when user hits the Send button, we will call the ``create`` mentioned earlier in the ``chats`` instance variable so that the chat message can be delivered to the Action Cable server. To do that, let's add an ``onClick`` listener to Send button as shown below:

```js
<button
  onClick={ (e) => this.handleSendEvent(e) }
  className='send'>
  Send
</button>
```

Then add the function ``handleSendEvent`` to handle the ``onClick`` event and do the message sending

```js
handleSendEvent(event) {
  event.preventDefault();
  this.chats.create(this.state.currentChatMessage);
  this.setState({
    currentChatMessage: ''
  });
}
```

In order to try out this, we will need to start this front end app using the following command:

```bash
npm start
```

and start the rails back end app using the command:

```bash
rails s -p 3001
```

When we try out to type something in the message box and click send, we will send some logs output in rails log as shown below:

```bash
ChatChannel#create({"content"=>"Hey there"})
   (0.1ms)  BEGIN
  SQL (0.4ms)  INSERT INTO `chat_messages` (`content`, `created_at`, `updated_at`) VALUES ('Hey there', '2017-07-30 04:10:12', '2017-07-30 04:10:12')
   (5.4ms)  COMMIchat_messageActiveJob] Enqueued ChatMessageCreationEventBroadcastJob (Job ID: 12699e81-4555-4ebd-b6eb-b072d4f2e495) to Async(default) with arguments: #<GlobalID:0x007ff3c99dc780 @uri=#<URI::GID gid://chat-app/ChatMessage/14>>
  ChatMessage Load (0.2ms)  SELECT  `chat_messages`.* FROM `chat_messages` WHERE `chat_messages`.`id` = 14 LIMIT 1
[ActiveJob] [ChatMessageCreationEventBroadcastJob] [12699e81-4555-4ebd-b6eb-b072d4f2e495] Performing ChatMessageCreationEventBroadcastJob (Job ID: 12699e81-4555-4ebd-b6eb-b072d4f2e495) from Async(default) with arguments: #<GlobalID:0x007ff3cc079898 @uri=#<URI::GID gid://chat-app/ChatMessage/14>>
[ActiveJob] [ChatMessageCreationEventBroadcastJob] [12699e81-4555-4ebd-b6eb-b072d4f2e495] [ActionCable] Broadcasting to chat_channel: {:chat_message=>"Hey there"}
[ActiveJob] [ChatMessageCreationEventBroadcastJob] [12699e81-4555-4ebd-b6eb-b072d4f2e495] Performed ChatMessageCreationEventBroadcastJob (Job ID: 12699e81-4555-4ebd-b6eb-b072d4f2e495) from Async(default) in 0.2ms
ChatChannel transmitting {"chat_message"=>"Hey there"} (via streamed from chat_channel)
```

This shows us that the message has successfully been sent to Rails Action Cable and the message is also broadcast to all the subscribers. If you look into JS console, you will see the broadcast data is also logged there:

<p style='text-align:center;' markdown='1'><img src='/public/images/chat_log.png' alt="Chat Log" style='display:inline;'/></p>

Our next task would be display all the chat logs. In order to do that, we would need to create a chat logs state and keep adding the received chat log into this state object.

First, we would need to modify the state object initialisation as shown below:

```js
this.state = {
  currentChatMessage: '',
  chatLogs: []
};
```

Then change the ``received`` function to push the chat log inside this state variable:

```js
// ...
received: (data) => {
  let chatLogs = this.state.chatLogs;
  chatLogs.push(data);
  this.setState({ chatLogs: chatLogs });
},
// ...
```

Then create a function ``renderChatLog`` to render out the list of chat messages inside ``chatLogs`` state variable:

```js
// ...
renderChatLog() {
  return this.state.chatLogs.map((el) => {
    return (
      <li key={`chat_${el.id}`}>
        <span className='chat-message'>{ el.content }</span>
        <span className='chat-created-at'>{ el.created_at }</span>
      </li>
    );
  });
}
// ...
```

Lastly, call this function inside ``render`` method to render the chat log:

```js
// ...
<ul className='chat-logs'>
  { this.renderChatLog() }
</ul>
// ...
```

We can improve the usability a little bit by allowing user to hit enter to submit the chat. This can be done by adding ``onKeyPress`` callback to ``input``

```js
// ...
<input
  onKeyPress={ (e) => this.handleChatInputKeyPress(e) }
  value={ this.state.currentChatMessage }
  onChange={ (e) => this.updateCurrentChatMessage(e) }
  type='text'
  placeholder='Enter your message...'
  className='chat-input' />
// ...
```

And add the function ``handleChatInputKeyPress``

```js
handleChatInputKeyPress(event) {
  if(event.key === 'Enter') {
    this.handleSendEvent(event);
  }//end if
}
```

And that's it. You can now start playing with the chat app:

<p style='text-align:center;' markdown='1'>
  <img src='/public/gifs/chat_rails_react.gif' alt="Chat App" style='display:inline;'/>
</p>

The source file for this article is made publicly available on the following URLs:

[https://github.com/jameshuynh/blog-codes/tree/master/rails-action-cable-react](https://github.com/jameshuynh/blog-codes/tree/master/rails-action-cable-react)
