---
layout: post
title: Getting Started with ReactJS
comments: true
excerpt_separator: <!-- more -->
---

ReactJS is a library created by Facebook. It is getting hot and more and more popular recently. One of the great companion of ReactJS is Webpack, which is tremendously help to speed up and ease the coding of ReactJS.

In this article, I would like to introduce how we can get started with the combination of Webpack and ReactJS. This would cover some basic setup and how we can start using to develop a web app straight away.

<!-- more -->

## Install Webpack

Before carrying on, let's make sure we have the latest nodejs. You can check your nodejs version by running the following command

{% highlight bash %}
node -v
{% endhighlight %}

At the point that I am writing this article, my nodejs version is ``5.6.0``. Hence, make sure your nodejs version is later than mine.

If your nodejs's version is ``0.10.xx`` or even lower, you should upgrade it. On Mac, you can trigger the following command to upgrade nodejs

{% highlight bash %}
brew update
brew install nodejs
{% endhighlight %}

Next stuff, let's install Yeoman by running the following command:

{% highlight bash %}
npm install -g yo
{% endhighlight %}

Following by install ``regenerator-react-webpack``

{% highlight bash %}
npm install -g generator-react-webpack
{% endhighlight %}

Finally, you can create a folder and create a react-webpack for your project:

{% highlight bash %}
mkdir my-new-project && cd my-new-project
yo react-webpack
{% endhighlight %}

The installer will ask you a few questions as following:

{% highlight bash %}
     _-----_
    |       |
    |--(o)--|   .--------------------------.
   `---------´  |    Welcome to Yeoman,    |
    ( _´U`_ )   |   ladies and gentlemen!  |
    /___A___\   '__________________________'
     |  ~  |
   __'.___.'__
 ´   `  |° ´ Y `

Out of the box I include Webpack and some default React components.

? Please choose your application name: myNewProject
? Which styles language you want to use? scss
? Enable postcss? No
{% endhighlight %}

You can experiment your answers. However, in the scope of this article, let's use the following setups:

- Please choose your application name: __myNewProject__
- Which styles language you want to use? __scss__
- Enable postcss? __No__

Once you are done with your answers, yeoman would carry on with all the necessary installation. After the installation, it should have the following folder structure:

{% highlight bash %}
├── cfg
├── dist
├── karma.conf.js
├── node_modules
├── package.json
├── server.js
├── src
├── test
└── webpack.config.js
{% endhighlight %}

## Running Webpack application

From the directory that we have done the installation, trigger the following command to run the application:

{% highlight bash %}
node server
{% endhighlight %}

After it runs, you should see a home page with Yeoman image in the middle

## Creating first ReactJS component

By default, the js source code is ES2015 code ([https://babeljs.io/docs/learn-es2015/](https://babeljs.io/docs/learn-es2015/)). You would probably need to get familar with this before jumping any development. But do not worry, athough the syntax is a bit different, its nature is still javascript.

The generator already creates an initial component for you which resides in ``src/components/Main.js``. If you would like to create a new ReactJS component, you can trigger the following command:

{% highlight bash %}
yo react-webpack:component my/namespaced/components/name
{% endhighlight %}

For instance, I can generate a Home component by issuing the following command:

{% highlight bash %}
yo react-webpack:component Home
{% endhighlight %}

This will generate ``src/components/HomeComponent.js``, which you can start coding inside that file

## Intall and use React Router

In a typical yeoman generator project, to manage javascript library, you will not need to manually download the source code and include into your project folder. Instead, you can use node package manager (npm) to perform the library dependencies. In this case, in order to install React Router, trigger the following command from your project folder

{% highlight bash %}
npm i --save react-router
{% endhighlight %}

This will perform the downloading of react-router library. At the same time, it will add a line inside your ``package.json``:

{% highlight bash %}
"react-router": "^2.0.0"
{% endhighlight %}

Once we have react-router library installed, we can replace the code inside ``src/index.js`` with the following code:

{% highlight javascript %}
import 'core-js/fn/object/assign';
import React from 'react';
import ReactDOM from 'react-dom';
import App from './components/Main';
import { Router, Route, Link, hashHistory } from 'react-router';

// Render the main component into the dom
// Comment the render code as App will now be rendered inside root Router below
// ReactDOM.render(<App />, document.getElementById('app'));

ReactDOM.render((
  <Router history={hashHistory}>
    <Route path="/" component={App} />
  </Router>
), document.getElementById('app'));
{% endhighlight %}

## Install and use jQuery

Same as install react-router library, we can easily install jQuery library by issuing the following command:

{% highlight bash %}
npm i --save jquery
{% endhighlight %}

After you can import jquery wherever you want to start using it:

{% highlight javascript %}
import $ from 'jQuery';
{% endhighlight %}

However, I feel quite inconvenient that every time I need to use jQuery, I would need to import. jQuery is a component that I would like to use on most of this site. If you have this kind of thought, then there is a way to automatically import jQuery everytime you call $ or jQuery. Let's move on to the next section

## Lazy load a component

You can set jQuery or any library to be lazy loaded by using ``webpack.ProvidePlugin``. This would need to be set inside ``cfg/dev.js`` like following.

{% highlight javascript %}
plugins: [
  new webpack.HotModuleReplacementPlugin(),
  new webpack.NoErrorsPlugin(),
  new BowerWebpackPlugin({
    searchResolveModulesDirectories: false
  }),

  // add the following lines
  // using webpack.ProvidePlugin to enable Lazy Load
  new webpack.ProvidePlugin({
    $: "jQuery",
    jQuery: "jQuery"
  })
],
{% endhighlight %}

You would also need to add in the same set of codes inside ``cfg/dist.js`` to enable to lazy load when distribute the folder as well:

{% highlight javascript %}
plugins: [
  new webpack.optimize.DedupePlugin(),
  new webpack.DefinePlugin({
    'process.env.NODE_ENV': '"production"'
  }),
  new BowerWebpackPlugin({
    searchResolveModulesDirectories: false
  }),
  new webpack.optimize.UglifyJsPlugin(),
  new webpack.optimize.OccurenceOrderPlugin(),
  new webpack.optimize.AggressiveMergingPlugin(),
  new webpack.NoErrorsPlugin(),

  // add the following lines
  // using webpack.ProvidePlugin to enable Lazy Load
  new webpack.ProvidePlugin({
    $: "jQuery",
    jQuery: "jQuery"
  })
],
{% endhighlight %}

After we have done with the setup, we would need to restart your ``node server`` by stop and start this process. Moving fowards, we would not need to import jQuery anymore, wherever we use either ``$`` or ``jQuery``, Webpack would automatically import jQuery for us.

## Conclusion

We have gone through the entire process after do basic initial setup with ReactJS, Webpack and Yeoman. After this setup, we would be able to start all the coding for the project.
