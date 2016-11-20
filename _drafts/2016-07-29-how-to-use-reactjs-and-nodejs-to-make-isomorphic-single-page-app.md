---
layout: post
title: How to use ReactJS and NodeJS to make Isomorphic Single Page App
date: 2016-07-29
comments: true
categories: [react, nodejs, isomorphic]
tags: [react, nodejs, ismorphic]
excerpt_separator: <!-- more -->
---

In my earlier post, I have mentioned about how we can use ``PhantomJS`` in order to achieve the ability to get crawled by a search engine optimisation. The reason why I wrote that post is because we have an app on BackboneJS and MarionetteJS that needs to be indexed by a search engine. ``PhantomJS`` would come in handy for that particular app as there would be no way BackboneJS and MarionetteJS component can be rendered on the server for the search engine. However, when we come to ReactJS, we have a choice now. Instead of rendering the ReactJS component on the browser, we can choose to render the ReactJS component on the server. With that, we can easily serve the content to a search engine without using ``PhantomJS``. In this post, I am going to go through how we can achieve the same result with the earlier post but with the help of NodeJS on frontend in order to make our app Isomorphic.

<!-- more -->

We are going to go through the following steps:

1. Setup a simple NodeJS app using ExpressJS (https://expressjs.com)
2. Create ReactJS Components
3. Render ReactJS Components on the server
4. Try to crawl the app using ``curl``

## 1. Setup a simple NodeJS app using ExpressJS

First and foremost, let create an empty folder and let's name it ``crawl-optimisation-web-app-nodejs``:

```bash
mkdir -p crawl-optimisation-web-app-nodejs
```

Next run ``npm init`` to initialize the folder as a NodeJS project:

```bash
cd crawl-optimisation-web-app-nodejs
npm init
```

The command will prompt you for quite a number of input but you can all set to default by hitting ENTER key until it finishes. After this, inside this folder, you should have a file called ``package.json``.

Next, install ``ExpressJS`` package as a dependency:

```bash
npm i express --save
```

Then you can create a file called ``app.js`` inside this folder with the following content:

```js
var express = require('express');
var app = express();

app.get('/', function (req, res) {
res.send('Hello World!');
});

app.listen(3000, function () {
  console.log('Listening on port 3000');
});
```

Save this file and go back to your terminal, you can now run the following command to run this app:

```bash
node app.js
```

Then now go to [http://localhost:3000](http://localhost:3000), you would see the message ``Hello World!`` there.

We are done with setting up the NodeJS app using ExpressJS. Let's move on the next task.

## 2. Create ReactJS Components

We do the same as what I did from my last article:

- A page to display a list of users.
- A page to display the detail of a particular user.

The content of ``public/index.html.ejs`` is as following

```html
<!-- public/index.html -->

<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Crawlable App</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react-dom.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/superagent/2.1.0/superagent.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.34/browser.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react-router/2.5.2/ReactRouter.min.js"></script>
  </head>
  <body>
    <div id="root"></div>
    <script type="text/babel" src="scripts/app.js"></script>
  </body>
</html>
```

The content of ``public/scripts/app.js`` is as following

```js
// public/scripts/app.js

let Router = ReactRouter.Router;
let Route = ReactRouter.Route;
let Redirect = ReactRouter.Redirect;
let backendURL = 'http://seo-backend.jameshuynh.com';
let browserHistory = ReactRouter.browserHistory;

class App extends React.Component {
  render() {
    return (
      <div className='app'>
        { this.props.children }
      </div>
    );
  }
}

class UsersIndex extends React.Component {

  constructor(props) {
    super(props);
    this.state = {
      allUsers: null
    };
  }

  componentWillMount() {
    superagent
      .get(backendURL + '/users')
      .end((err, res) => {
        if(!err) {
          this.setState({ allUsers: res.body });
        }//end if
      });
  }

  render() {
    if(this.state.allUsers) {
      let users = this.state.allUsers.map((user) => {
        return (
          <tr key={ user.id }>
            <td>{ user.id }</td>
            <td>{ user.name }</td>
            <td>{ user.title }</td>
            <td><a href={`/users/${user.id}`}>View</a></td>
          </tr>
        );
      });

      return (
        <table>
          <thead>
            <tr>
              <th>ID</th>
              <th>Name</th>
              <th>Title</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            { users }
          </tbody>
        </table>
      );
    } else {
      return (
        <div>Loading...</div>
      );
    }
  }
}

class UserDetail extends React.Component {
  constructor(props) {
    super(props);
    this.state = { user: null };
  }

  componentWillMount() {
    superagent
      .get(backendURL + '/users/' + this.props.params.userId)
      .end((err, res) => {
        if(!err) {
          this.setState({ user: res.body });
        }
      });
  }

  render() {
    if(this.state.user) {
      let user = this.state.user;
      return (
        <div>
          <ul>
            <li>ID: {user.id}</li>
            <li>Name: {user.name}</li>
            <li>Title: {user.title}</li>
            <li>Bio: {user.bio}</li>
          </ul>
          <a href='/users'>Back</a>
        </div>
      );
    } else {
      return (<div>Loading...</div>);
    }
  }
}

ReactDOM.render((
  <Router history={browserHistory}>
    <Redirect from='/' to='/users' />
    <Route path="/" component={App}>
      <Route path="/" component={UsersIndex}/>
      <Route path="/users" component={UsersIndex}/>
      <Route path="/users/:userId" component={UserDetail} />
    </Route>
  </Router>
), document.getElementById('root'));
```

