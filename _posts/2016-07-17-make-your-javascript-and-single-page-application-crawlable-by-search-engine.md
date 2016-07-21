---
layout: post
title: Make your javascript application and single page application crawlable by search engine
date: 2016-07-17
comments: true
categories: [seo, rails, spa, javascript]
tags: [seo, rails, spa, javascript]
excerpt_separator: <!-- more -->
---

With the growth of javascript frameworks and libraries like ReactJS, AngularJS, EmberJS, etc..., javascript application is a trend now. A lot of developers have switched from using a Rails, PHP or Java to do frontend application to use these mentioned javascript frameworks and libraries. However, there is a major drawback by using these framework, that is your frontend application would not be easily crawled by a search engine like Google, Yahoo or Bing. In this article, I would like to outline a solution that I have experimented on a javascript app to make the frontend app crawlable regardless of which javascript frameworks or libraries that are used.

<!-- more -->

## Tools

In order to perform these tasks, here are the tools that I am going to use to outline the solution:

- nginx (or apache) ([http://nginx.org](http://nginx.org))
- Rails (or any backend language that you are comfortable) ([http://rubyonrails.org](http://rubyonrails.org))
- ReactJS (or any web frontend framework) ([https://facebook.github.io/react/](https://facebook.github.io/react/))
- ``dynamic_sitemaps`` Ruby Gem ([https://rubygems.org/gems/dynamic_sitemaps](https://rubygems.org/gems/dynamic_sitemaps))
- ``rack-cors`` Gem ([https://rubygems.org/gems/rack-cors](https://rubygems.org/gems/rack-cors))
- PhantomJS ([http://phantomjs.org](http://phantomjs.org))

## Overview of the data flow

<p style='text-align:center;' markdown='1'><img src='/public/images/seo_diagram.svg' style='display:inline;'/></p>

- The purple lines are lines to indicate the data flow of a request from a search bot.
- The brown lines are lines to indicate how the app can prepapre the response from search bot request.
- The green lines are lines to indicate how a normal end user browser would hit the frontend javascript application.

## Steps that I am going to perform

1. Prepare Rails application with some user seeds data and a controller to return user listing and detail data.
2. Make the Rails app allow the frontend application to call the API cross origin using ``rack-cors`` gem.
3. Prepare a simple 2-page Javascript application which will call the user listing API and to display the user listing and call the user detail API to display the user detail.
4. Try to crawl the Javacript application using ``curl`` command.
5. Develop a ``PhantomJS`` script and crawl the Javascript application using ``PhantomJS``.
6. Develop a controller action to receive the request from search engine, then use ``PhantomJS`` to crawl the requested URL or use the cached content inside Redis DB to respond back to the request.
7. Use ``nginx`` to route the search engine request to Rails server.
8. Develop a sitemap script to generate sitemap containing the users index URL and each user detail URL.
9. Develop a sitemap reader script to read the sitemap script and return all the URL specified in the sitemap script.
10. Make a Rake task to use the sitemap reader script in order to get all sitemap URLs. Crawl contents of each of the URLs insite the sitemap URLs and store the content inside Redis DB using the URL path as the key.
11. Schedule the Rake task to run at 1 am every day.

## 1. Prepare Rails application with some user seeds data and a controller to return user listing and detail data.

To start, let create a Rails app by issuing the following command:

```bash
rails new backend
```

Then create a User model by issuing the following command:

```bash
rails g model User name title bio:text
```

Prepare some seeds data

```ruby
## db/seeds.rb

[
  {
    name: 'James Huynh',
    title: 'MR',
    bio: 'I am a Rails & Front End Developer'
  },
  {
    name: 'Ivy Pham',
    title: 'MS',
    bio: 'I am a manager assistant'
  }
].each do |user_data|
  User.create(user_data)
end
```

Then generate a controller for users index and show API:

```bash
rails g controller users
```

And make the ``users_controller.rb`` have the following code

```ruby
class UsersController < ApplicationController
  def index
    render json: User.all
  end

  def show
    render json: User.find_by_id(params[:id])
  end
end
```

And finally add in ``routes.rb`` for user resource with ``index`` and ``show`` action only:

```ruby
resources :users, only: [:index, :show]
```

## 2. Make the Rails app allow the frontend application to call the API cross origin using ``rack-cors`` gem.


To enable the frontend to call the API in this rails backend, we would need to add ``rack-cors`` gem tell Rails to allow cross origin request. Let's do this by adding the following line inside your ``Gemfile``:

```ruby
gem 'rack-cors', :require => 'rack/cors'
```

And add the following codes into ``application.rb`` to enable all JS requests from all other domains. You can restrict the allowed domains as what you want.

```ruby
config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'
    resource '*', :headers => :any, :methods => [:get, :post, :options]
  end
end
```

I then deploy this rails app into [http://seo-backend.jameshuynh.com](http://seo-backend.jameshuynh.com).

## 3. Prepare a simple 2-page Javascript application which will call the user listing API and to display the user listing and call the user detail API to display the user detail.

To start, let's create 2 page javascript application. For the purpose of this blog, I am going to use ReactJS to develop the this demo application. However, this solution is not bound to only ReactJS app. You can use this solution in any kind of javascript libraries or frameworks that you are using:

These 2 pages will be:

- A page to display a list of users.
- A page to display the detail of a particular user.

This app has only 2 files and 1 ``public`` folder:

The content of ``public/index.html`` is as following

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

I then deploy this frontend app into [http://seo-frontend.jameshuynh.com](http://seo-frontend.jameshuynh.com)

Then I can access [http://seo-frontend.jameshuynh.com](http://seo-frontend.jameshuynh.com) and should see the followings:

<p style='text-align:center;' markdown='1'><img src='/public/images/reactfront1.jpg' srcset='/public/images/reactfront1@2x.jpg 2x' style='display:inline;'/></p>

Clicking on View link would bring user to the next page, which is the user detail page:

<hr />
<p style='text-align:center;' markdown='1'><img src='/public/images/reactfront2.jpg' srcset='/public/images/reactfront2@2x.jpg 2x' style='display:inline;'/></p>

## 4. Try to crawl the Javacript application using ``curl`` command.

Now if we try to crawl this frontend app using the following command. The output will be as following:

```bash
$ curl http://seo-frontend.jameshuynh.com
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

Apparently, we would not be able to crawl the page content as the page content is generated by javascript and ``crawl`` will not be able to understand javascript. This result will not do any interest to search engine as there is NO content like what we saw earlier when we load the web page.

## 5. Develop a ``PhantomJS`` script and crawl the Javascript application using ``PhantomJS``.

What are we going to try out is to use ``PhantomJS`` to try to crawl the page. To install ``PhantomJS`` on Linux server, you can visit the following URL [http://phantomjs.org/build.html](http://phantomjs.org/download.html) on Linux or ``brew install phantomjs`` on Mac OSX

Next, in your rails project, create a file inside ``lib`` folder called ``phantomjs-crawler.js`` with the following JS code:

```js
var page = require('webpage').create();
var system = require('system');

var lastReceived = new Date().getTime();
var requestCount = 0;
var responseCount = 0;
var requestIds = [];
var startTime = new Date().getTime();

page.onResourceReceived = function (response) {
    if(requestIds.indexOf(response.id) !== -1) {
        lastReceived = new Date().getTime();
        responseCount++;
        requestIds[requestIds.indexOf(response.id)] = null;
    }
};
page.onResourceRequested = function (request) {
    if(requestIds.indexOf(request.id) === -1) {
        requestIds.push(request.id);
        requestCount++;
    }
};

// Open the page
page.open(system.args[1], function () {});

var checkComplete = function () {

  // we allow max 2 seconds to evaluate the last script
  // or MAX 10 seconds for the entire site
  if((new Date().getTime() - lastReceived > 2000 &&
     requestCount === responseCount) ||
       new Date().getTime() - startTime > 10000)  {
    clearInterval(checkCompleteInterval);
    console.log(page.content);
    phantom.exit();
  }
}

/// Let us check to see if the page is finished rendering
var checkCompleteInterval = setInterval(checkComplete, 1);
```

This is a standard ``PhantomJS`` web page crawling script. We start by creating a ``webpage`` object. We then let the webpage object to listen to 2 events ``onResourceReceived`` and ``onResourceRequested``. After all, we set the interval for checking if the site has fully loaded by calling ``setInterval`` and put in a condition for cut over of the web page content. You can always adjust the condition inside ``checkComplete`` function to suit your site content. After the cut over happens, we then print out the web page content and exit ``PhantomJS`` script.

Now if I run the following command from your Rails root, I would see the listing of the users is now returned fully.

```bash
$ phantomjs lib/phantomjs-crawler.js http://seo-frontend.jameshuynh.com/users

<!-- public/index.html --><!DOCTYPE html><html><head>
    <meta charset="utf-8">
    <title>Crawlable App</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react-dom.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.34/browser.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react-router/2.5.2/ReactRouter.min.js"></script>
  </head>
  <body>
    <div id="root"><div data-reactroot="" class="app"><table><thead><tr><th>ID</th><th>Name</th><th>Title</th><th></th></tr></thead><tbody><tr><td>1</td><td>James Huynh</td><td>MR</td><td><a href="#/users/1">View</a></td></tr><tr><td>2</td><td>Ivy Pham</td><td>MS</td><td><a href="#/users/2">View</a></td></tr></tbody></table></div></div>
    <script type="text/babel" src="scripts/app.js"></script>


</body></html>
```

This is much better now as the content that we want has been returned accurately now. If I now try out with the view URL, I would be able to see the correct page content as well:

```bash
$ phantomjs lib/phantomjs-crawler.js http://seo-frontend.jameshuynh.com/users/1

<!-- public/index.html --><!DOCTYPE html><html><head>
    <meta charset="utf-8">
    <title>Crawlable App</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react-dom.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.23/browser.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react-router/2.5.2/ReactRouter.min.js"></script>
  </head>
  <body>
    <div id="root"><div data-reactroot="" class="app"><div><ul><li><!-- react-text: 5 -->ID: <!-- /react-text --><!-- react-text: 6 -->2<!-- /react-text --></li><li><!-- react-text: 8 -->Name: <!-- /react-text --><!-- react-text: 9 -->Ivy Pham<!-- /react-text --></li><li><!-- react-text: 11 -->Title: <!-- /react-text --><!-- react-text: 12 -->MS<!-- /react-text --></li><li><!-- react-text: 14 -->Bio: <!-- /react-text --><!-- react-text: 15 -->I am a manager assistant<!-- /react-text --></li></ul><a href="#/users">Back</a></div></div></div>
    <script type="text/babel" src="scripts/app.js"></script>


</body></html>
```

## 6. Develop a controller action to receive the request from search engine, then use ``PhantomJS`` to crawl the requested URL or use the cached content inside Redis DB to respond back to the request.

Although ``PhantomJS`` can crawl the page's content easily, we will not let search engine directly hit the ``PhantomJS`` as the crawling is usually slow and can result in a time out from the search engine.

What I want to do now is calling this ``PhantomJS`` program from our Rails app periodically and store the content into Redis server. When a search engine tries to crawl our frontend application, we would direct the request to our Rails app and serve the cached crawling content there. In case, the content is not found in the cache, we would then perform a crawl using ``PhantomJS`` and cache it into Redis before sending back the response to Search Engine.

Back to our Rails app, add ``redis-namespace`` gem into ``Gemfile``. This gem will help us create a namespace to store all our crawling content.

```ruby
gem 'redis-namespace', '~> 1.5.2'
```

Then run bundle install:

```ruby
bundle install
```

Now create a controller called ``seo``:

```bash
rails g controller seo
```

Inside this ``seo_controller.rb``, paste in the following code:

```ruby
class SeoController < ApplicationController
  layout false

  def content

    # get the path from the request
    frontend_full_path = params[:path]

    ## initialize a redis instance
    connection = Redis.new(:url => "redis://127.0.0.1/12")
    redis = Redis::Namespace.new(:seo, :redis => connection)

    # try to get out the cached content
    html = redis.get(frontend_full_path)
    if html.nil?
      # no cache content - crawl using PhantomJS
      html = crawl_frontend(frontend_full_path)
    end

    # respond back to search engine
    render inline: html
  end

  private

  def crawl_frontend(fullpath)
    # perform a crawl
    response = %x{ phantomjs --ssl-protocol=any #{Rails.root}/lib/phantomjs-crawler.js "http://seo-frontend.jameshuynh.com/#{fullpath}" }

    # initialize a redis instance
    connection = Redis.new(:url => "redis://127.0.0.1/12")
    redis = Redis::Namespace.new(:seo, :redis => connection)

    # store the crawling content using fullpath as the key
    redis.set(fullpath, response)

    return response
  end
end
```

Now add in a route to route to this action inside Rails app - ``routes.rb``

```ruby
get '/seo', controller: :seo, action: :content
```

As what I said earlier, we will somehow redirect the request from search engine to Rails app ``seo#content``  and pass the ``path`` parameter that it intends to request to this backend rails through ``path`` params. In ``content`` action, we then try to get the cache content from redis using the ``path`` as the key. If that could not be found, we would then perform a crawl using ``phantomjs`` in order to get the content before returning back the response to search engine.

Now, I redeploy the Rails app with the ``seo`` controller code and do some experiment. Let's issue the following ``curl`` request:

```bash
$ curl "http://seo-backend.jameshuynh.com/seo?path=/users"

<!-- public/index.html --><!DOCTYPE html><html><head>
    <meta charset="utf-8">
    <title>Crawlable App</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react-dom.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.23/browser.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react-router/2.5.2/ReactRouter.min.js"></script>
  </head>
  <body>
    <div id="root"><div data-reactroot="" class="app"><table><thead><tr><th>ID</th><th>Name</th><th>Title</th><th></th></tr></thead><tbody><tr><td>1</td><td>James Huynh</td><td>MR</td><td><a href="#/users/1">View</a></td></tr><tr><td>2</td><td>Ivy Pham</td><td>MS</td><td><a href="#/users/2">View</a></td></tr></tbody></table></div></div>
    <script type="text/babel" src="scripts/app.js"></script>


</body></html>
```

I should see the same content as what we did earlier when we were trying to crawl the URL [http://seo-frontend.jameshuynh.com/users](http://seo-frontend.jameshuynh.com/users). As you would notice, this ``curl`` request takes sometime to finish. However, subsequently, when I issue this curl request again, it would be extremely fast, as it has served the content cached inside Redis DB. This shows that our Rails app now can be used to serve the search engine request efficiently.

The only missing piece now is to direct the traffic from search engine to our rails app everytime it tries to hit our frontend application. In order to do that we would need to configure our proxy server, which in this case, I am going to show how to do that with nginx server.

## 7. Use ``nginx`` to route the search engine request to Rails server.

In order to route the request, we can use ``$http_user_agent`` to detect the search engine agent. If it is one of the following agents: ``curl``, ``wget``, ``googlebot``, etc..., we would then rewrite the URL to be ``/seo?path=<initial_url>`` and then ``proxy_pass`` it to the declared upstream Rails. The partial nginx code belows illustrates how we can do that:

__Note__: We are using ``set_escape_uri`` function in nginx config, which you would probably needs to compile nginx with this module [https://github.com/openresty/set-misc-nginx-module#set_escape_uri](https://github.com/openresty/set-misc-nginx-module#set_escape_uri]).

```bash
upstream seo_backend {
  server unix:/tmp/seo_backend.sock fail_timeout=0;
}

...

server {
  listen app-server:80;
  server_name seo-frontend.jameshuynh.com;
  root /home/app/www/seo/frontend/current/public;

  location ~ ^/(images|scripts|javascripts|stylesheets|system)/ {
    root /home/app/www/seo/frontend/current/public;
  }

  location / {

    if ($request_uri ~* "sitemap") {
      break;
    }

    if ($http_user_agent ~* "curl|Wget|googlebot|Google\-Structured\-Data\-Testing\-Tool|yahoo|bingbot|baiduspider|yandex|yeti|yodaobot|gigabot|ia_archiver|facebookexternalhit|twitterbot|developers\.google\.com") {
      set_escape_uri $escape_request_uri $request_uri;
      set $args path=$escape_request_uri;
      rewrite ^(.*)$ /seo;
      proxy_pass http://seo_backend;
      break;
    }

    rewrite .* /index.html break;
  }

  access_log  /var/log/nginx/seo_frontend.log;
  error_log  /var/log/nginx/seo_frontend.log;
}
```

After putting this nginx configuration. We can then try again with the following ``curl`` command:

```bash
$ curl "http://seo-frontend.jameshuynh.com/users"

<!-- public/index.html --><!DOCTYPE html><html><head>
    <meta charset="utf-8">
    <title>Crawlable App</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.2.1/react-dom.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.23/browser.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react-router/2.5.2/ReactRouter.min.js"></script>
  </head>
  <body>
    <div id="root"><div data-reactroot="" class="app"><table><thead><tr><th>ID</th><th>Name</th><th>Title</th><th></th></tr></thead><tbody><tr><td>1</td><td>James Huynh</td><td>MR</td><td><a href="#/users/1">View</a></td></tr><tr><td>2</td><td>Ivy Pham</td><td>MS</td><td><a href="#/users/2">View</a></td></tr></tbody></table></div></div>
    <script type="text/babel" src="scripts/app.js"></script>


</body></html>
```

This is much better now as you can see now we can directly ``curl`` the front end URL and yet it is returning the correct data.

However, this is still not ideal, as search engine can hit a page and it can be extremely slow, which can result in the fact that serch engine will refuse to crawl that page again in the future.

What we probably want to do now is that setup a ``sitemap`` and submit the ``sitemap`` to search engine for indexing. At the same time, we would let ``PhamtomJS`` crawl all the URLs indicated in the ``sitemap`` periodically and cache the content in our Redis DB. This way, when search engine indexes all the pages inside our ``sitemap``, it will always hit our Redis DB cache.

## 8. Develop a sitemap script to generate sitemap containing the users index URL and each user detail URL.

To start for this task, let add the ``dyanmic_sitemap`` gem in our ``Gemfile``:

```ruby
gem 'dynamic_sitemaps', '~> 2.0.0'
```

Then create a file named ``sitemap.rb`` inside Rails ``config`` folder with the following code:

```ruby
front_end_url = "seo-frontend.jameshuynh.com"

host front_end_url

sitemap :site do
  url "http://#{front_end_url}/", last_mod: Time.now, change_freq: "daily", priority: 1.0
  url "http://#{front_end_url}/users", priority: 0.95
  url "http://#{front_end_url}/users/", priority: 0.95
end

sitemap_for User.all, name: :user do |user|
  url "http://#{front_end_url}/users/#{user.id}",
      last_mod: user.updated_at,
      priority: 0.95
end

ping_with "#{front_end_url}/sitemap.xml"
```

We would then need to create a Rake task to generate the sitemap and symlink with the frontend app folder. Create a file named ``lib/tasks/generate_sitemap.rake`` inside Rails ``lib`` folder with the following code.

```ruby
# lib/tasks/generate_sitemap.rake

task :generate_sitemap => :environment do
  Rake::Task["sitemap:generate"].invoke
  %x{ ln -nfs #{Rails.root}/public/sitemaps/sitemap.xml #{Rails.root}/public/sitemap.xml}
  %x{ ln -nfs #{Rails.root}/public/sitemaps/sitemap.xml #{Rails.root.to_s.gsub('backend', 'frontend').gsub(/releases\/\d+/, 'current')}/public/sitemap.xml}
  %x{ ln -nfs #{Rails.root}/public/sitemaps #{Rails.root.to_s.gsub('backend', 'frontend').gsub(/releases\/\d+/, 'current')}/public/sitemaps}
end
```

When run this task, it will generate the sitemap using ``dynamic_sitemap`` Rake task - ``rake sitemap:generate`` as well symlink the ``sitemap.xml`` and ``sitemaps`` folder into the frontend folder so that the sitemap.xml would be availabel on (http://seo-frontend.jameshuynh.com/sitemap.xml)[http://seo-frontend.jameshuynh.com/sitemap.xml]


## 9. Develop a sitemap reader script to read the sitemap script and return all the URLs specified in the sitemap script.

Next, we need to create a file named ``sitemap_reader.rb`` in order to consume ``sitemap.rb`` in order to get all the URLs churned out by this ``sitemap.rb``. I then create file ``sitemap_reader.rb`` inside ``lib`` folder

```ruby
# lib/sitemap_reader.rb

class SitemapReader
  def initialize(config_path="config/sitemap.rb")
    @urls = []
    self.instance_eval(File.open("#{Rails.root}/#{config_path}").read)
  end

  def host(url=nil)
    url
  end

  def sitemap(sym)
    @urls << yield
  end

  def sitemap_for(entities, opts={})
    entities.each do |entity|
      @urls << yield(entity)
    end
  end

  def url(url, opts={})
    url.gsub(/\s/, '')
  end

  def urls
    @urls
  end

  def ping_with(url)
  end
end
```

We will use this Ruby class in our next step.

## 10. Make a Rake task to use the sitemap reader script in order to get all sitemap URLs. Crawl contents of each of the URLs insite the sitemap URLs and store the content inside Redis DB using the URL path as the key.

Now, we would need to create a Rake task to run periodically. This task would be used to get all the URLs in ``sitemap.rb``, then submit each of the URL to ``phantomjs`` and store the returned content inside ``redis``. To do that, create tasks named ``crawl_and_cache_site.rake`` inside ``lib/tasks/`` folder with the following content:

```ruby
## lib/tasks/crawl_and_cache_site.rake

require 'sitemap_reader'

task :crawl_and_cache_site => :environment do
  sitemap_reader = SitemapReader.new
  sitemap_reader.urls.each do |url|
    connection = Redis.new(:url => "redis://127.0.0.1/12")
    redis = Redis::Namespace.new(:seo, :redis => connection)
    fullpath = url.gsub('http://seo-frontend.jameshuynh.com', '')

    response = %x{ phantomjs --ssl-protocol=any #{Rails.root}/lib/phantomjs-crawler.js #{url} }
    redis.set(fullpath, response)
  end
end
```

I then try to run this task by issuing the following command:

```bash
$ bundle exec rake crawl_and_cache_site
```

## 11. Schedule the Rake task to run at 1 am every day.

Then schedule this task by using ``whenever`` gem inside ``Gemfile``:

```ruby
gem 'whenever', '~> 0.9.7'
```

Then bundle install

```bash
bundle install
```

And generate ``whenever``'s ``schedule.rb`` file by running this command:

```bash
bundle exec wheneverize
```

Then go to ``config/schedule.rb`` and add the following schedule:

```ruby
# generate & symlink sitemap at 0am every day
every 1.day, at: '0am' do
  rake 'generate_sitemap'
end

# refresh all the cached content of crawled pages at 1am everyday
every 1.day, at: '1am' do
  rake 'crawl_and_cache_site'
end
```

On the server, I will then run the ``whenever`` command from the Rails current deployed folder:

```bash
$ bundle exec whenever

0 1 * * * /bin/bash -l -c 'cd /path/to/backend/current && RAILS_ENV=production bin/rake crawl_and_cache_site --silent'

0 0 * * * /bin/bash -l -c 'cd /path/to/backend/current && RAILS_ENV=production bin/rake generate_sitemap --silent'
```

Finally, I will then copy the 2 generated lines and run ``crontab -e`` and paste at the end of the crontab and save it.

We are done!

## Summary

By completing all these tasks, our frontend site is now indexable by search engine. Congratulations!!! You have gone to this far. I hope you enjoy the article and please send me any questions or suggestions that you have. I would welcome all of them.

As usual, the source file for this article is made publicly available on the following URLs:

- Rails Backend
  - Live URL: [http://seo-backend.jameshuynh.com/users](http://seo-backend.jameshuynh.com/users)
  - Source code: [https://github.com/jameshuynh/crawl-optimisation-web-app-backend](https://github.com/jameshuynh/crawl-optimisation-web-app-backend)
  - Nginx configuration file: [https://github.com/jameshuynh/crawl-optimisation-web-app-backend/blob/master/nginx/sites-available/seo-backend-and-seo-frontend.jameshuynh.com](https://github.com/jameshuynh/crawl-optimisation-web-app-backend/blob/master/nginx/sites-available/seo-backend-and-seo-frontend.jameshuynh.com) 

- Javascript Frontend
  - Live URL: [http://seo-frontend.jameshuynh.com/users](http://seo-frontend.jameshuynh.com/users)
  - Source code: [https://github.com/jameshuynh/crawl-optimisation-web-app](https://github.com/jameshuynh/crawl-optimisation-web-app)

