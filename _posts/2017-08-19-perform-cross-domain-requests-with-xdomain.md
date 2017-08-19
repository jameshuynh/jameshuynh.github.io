---
layout: post
title: Perform cross domain request with xdomain
date: 2017-08-19
comments: true
categories: [rails, react js, cross domain]
tags: [rails, react js, cross domain]
excerpt_separator: <!-- more -->
---

We use a lot of cross domain requests from ReactJS app to Rails. At the moment, in order to do that, we would need to use [rack-cors](https://github.com/cyu/rack-cors), a gem to let Rails app to allow cross domain requests from javascript. I discovered [xdomain](https://github.com/jpillora/xdomain) recently and recognise that althought it is relatively a new tool to help to achieve the same thing but with simpler setup. In this article, I am going through on how we are going to setup and use this tool.

<!-- more -->

## 1. Prepare the Rails app with API only

Create a new Rails app with API mode

```bash
rails new xdomain-rails --api
```

Next, create a controller and an action:

```bash
rails g controller home welcome
```

Now, we would need to a create a small file called ``proxy.html`` inside our Rails public folder:

```bash
cd xdomain-rails
echo '<!DOCTYPE HTML><script src="//cdn.rawgit.com/jpillora/xdomain/0.7.4/dist/xdomain.min.js" master="http://localhost:8000"></script>' > public/proxy.html
```

``http://localhost:8000`` is the URL of our React App that we are going to create.

__Note__: You would need to swap ``proxy.html`` in your deploy script when you deploy to different environments. For instance, for production use, you would need to swap ``proxy.html`` with the same file but replace ``http://localhost:8000`` with the actual production URL.

Inside ``home_controller.rb``, let fill in a dummy json return for ``welcome`` action:

```ruby
# app/controllers/home_controller.rb
class HomeController < ApplicationController
  def welcome
    render json: { content: 'Welcome to my site' }
  end
end
```

## 2. Create ReactJS App

Let install ``create-react-app`` in order to create a react app easily

```bash
npm install -g create-react-app
```

Then create a React app by issuing a command:

```bash
create-react-app xdomain-app
```

To make our React Project run on port ``8000`` instead of the default port ``3000``, create ``.env`` and add in the line:

```
PORT=8000
```

We would also need to have the base API URL in this ``.env`` file so that it can be referred to in the app:

```
PORT=8000
REACT_APP_API_BASE_URL=http://localhost:3000
```

Next, add ``axios`` library to make ajax call to our Rails app:

```bash
yarn add axios
```

We would need to create an axiosClient to ensure that the API base URL is pointing to the Rails server. To do that, create a file src/axiosClient.js with the following content:


```js
// src/axiosClient.js
let axios = require('axios');

let axiosClient = axios.create({
  baseURL: process.env.REACT_APP_API_BASE_URL
});

export default axiosClient;
```

Next, modify current ``App.js`` to make an ajax call to Rails ``HomeController#welcome`` inside ``componentWillMount``


```js
import React, { Component } from 'react';
import logo from './logo.svg';
import axiosClient from './axiosClient';
import './App.css';

class App extends Component {
  constructor(props) {
    super(props);

    this.state = {
      message: 'Welcome to React'
    };
  }

  componentWillMount() {
    axiosClient.get('/home/welcome').then(response => {
      this.setState({
        message: response.data.content
      });
    });
  }

  render() {
    let { message } = this.state;

    return (
      <div className="App">
        <div className="App-header">
          <img src={logo} className="App-logo" alt="logo" />
          <h2>
            {message}
          </h2>
        </div>
      </div>
    );
  }
}

export default App;
```

Finally, we would need to add the magic line to ``index.html``:

```html
<head>
<!-- ... -->
<script
  src="//cdn.rawgit.com/jpillora/xdomain/0.7.4/dist/xdomain.min.js"
  slave="%REACT_APP_API_BASE_URL%/proxy.html"></script>
<!-- ... -->
```

After this, you can run both Rails app and React App. React App would be able to retrieve the content output inside ``HomeController#welcome`` and display in the React App

As usual, the source file for this article is made publicly available on the following URLs:

[https://github.com/jameshuynh/blog-codes/tree/master/reactjs-nested-form](https://github.com/jameshuynh/blog-codes/tree/master/reactjs-nested-form)

