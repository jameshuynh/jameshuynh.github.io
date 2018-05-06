---
layout: post
title: How to use graphql gem in Rails with ReactJS
date: 2018-05-06
comments: true
categories: [rails, reactjs, graphql]
tags: [rails, reactjs, graphql]
excerpt_separator: <!-- more -->
---

In this article, let's explore on how to use GraphQL implementation with Rails and ReactJS to build a simple app which would do the followings:

* Login with an email & password using graphql
* List all books with title, author name & number of comments

We would also explore how to deal with N+1 query problem with GraphQL

<!-- more -->

## Setup Rails App

First, let's create a Rails app with API only using MySQL as the database:

```bash
rails new graphql_book_app --database=mysql --api
```

After Rails has successfully created and bundle install the project, add in the followings into `Gemfile`:

```rb
gem 'bcrypt', '~> 3.1.11' # is needed for has_secure_password
gem 'graphiql-rails', '~> 1.4.10', group: :development
gem 'graphql', '~> 1.7.14'
gem 'graphql-preload', '~> 1.0.4' # is needed to fix N + 1 query issue
gem 'jwt', '~> 2.1.0'
gem 'rack-cors', '~> 1.0.2'
```

And then run bundle install again:

```bash
bundle install
```

After bundle installation, trigger this command to generate some graphql redefined files:

```bash
rails generate graphql:install
```

Next, let's generate the models that we need for this project:

```bash
rails g model user email password_digest
rails g model author name
rails g model book title author_id
rails g model comment book_id:integer content:text
```

Then add the following line into `user.rb`:

```rb
class User < ApplicationRecord
  has_secure_password
end
```

We have generated a field called `password_digest` for users table and add `has_secure_password` to `user.rb` so that an encrypted password could be generated and used for authentication. To create a user from rails console with a password, simple trigger the following from rails console:

```rb
user = User.create(email: 'admin@test.com', password: 'secret')
# then verify if the created user can be authenticated
user.authenticate('secret') # => should return user object
```

When calling authenticate with a correct password, the same user object would be returned. Otherwise, a `nil` object would be returned if supplied password is incorrect.

Next, we define the relationships for book model:

```rb
class Book < ApplicationRecord
  belongs_to :author
  has_many: comments, dependent: :destroy
end
```

and for comment model:

```rb
class Comment < ApplicationRecord
  belongs_to :book
end
```

and for author model:

```rb
class Author < ApplicationRecord
  has_many :books
end
```

Next, we will create some seeds data in `seeds.rb`

```rb
User.create(email: 'admin@example.com', password: 'secret')

author1 = Author.create(name: 'Drew Neil')
author2 = Author.create(name: 'Rob Isenberg')
book1 = Book.create(title: 'Modern Vim', author: author1)
book2 = Book.create(title: 'Docker for Rails Developers', author: author2)

book1.comments.create(content: 'This book is great')
book1.comments.create(content: 'This book is awesome')
book1.comments.create(content: 'I love this book')

book2.comments.create(content: 'Awesome book')
book2.comments.create(content: 'Truly powerful')
book2.comments.create(content: 'Beautiful content')
```

## Define all GraphQL Types & Query Types

Now we will need to define all the grapql types by creating the following files under `app/graphql/types/` folder:

* book_type.rb
* author_type.rb
* comment_type.rb

and create a file call `sign_in_user.rb` under `app/graphql/mutations` folder:

This is how the code will look like for these files:

```rb
# app/graphql/types/author_type.rb

require 'graphql/batch'
Types::AuthorType = GraphQL::ObjectType.define do
  name 'Author'

  field :id, !types.ID
  field :name, !types.String
end
```

```rb
# app/graphql/types/book_type.rb

require 'graphql/batch'
Types::BookType = GraphQL::ObjectType.define do
  name 'Book'

  field :id, !types.ID
  field :title, !types.String

  field :author, Types::AuthorType do
    resolve lambda { |obj, _args, _ctx|
      obj.author
    }
  end

  field :comments, Types::Comment do
    resolve lambda { |obj, _args, _ctx|
      obj.comments
    }
  end

  field :number_of_comments, !types.Int do
    resolve lambda { |obj, _args, _ctx|
      obj.comments.count
    }
  end
end
```

```rb
# app/graphql/types/comment_type.rb

require 'graphql/batch'
Types::CommentType = GraphQL::ObjectType.define do
  name 'Comment'

  field :id, !types.ID
  field :content, !types.String
end
```

We would also need to define the query for books by adding the code for ``query_type.rb`` like followings:

```rb
# app/graphql/types/query_type.rb

Types::QueryType = GraphQL::ObjectType.define do
  name 'Query'

  field :books, !types[Types::BookType] do
    resolve lambda { |_obj, _args, ctx|
      Book.all
    }
  end
end
```

To verify if we can query books, we can open rails console and run the followings:

```rb
query = %(query {
    books {
      id,
      title,
      comments {
        id,
        content
      },
      author {
        id,
        name
      }
    }
  }
)
result = GraphqlBookAppSchema.execute(query)
```

The following should appear

```
  Book Load (0.3ms)  SELECT `books`.* FROM `books`
  Comment Load (0.3ms)  SELECT `comments`.* FROM `comments` WHERE `comments`.`book_id` = 1
  Author Load (0.3ms)  SELECT  `authors`.* FROM `authors` WHERE `authors`.`id` = 1 LIMIT 1
  Comment Load (0.2ms)  SELECT `comments`.* FROM `comments` WHERE `comments`.`book_id` = 2
  Author Load (0.2ms)  SELECT  `authors`.* FROM `authors` WHERE `authors`.`id` = 2 LIMIT 1
=> #<GraphQL::Query::Result @query=... @to_h={"data"=>{"books"=>[{"id"=>"1", "title"=>"Modern Vim", "comments"=>[{"id"=>"1", "content"=>"This book is great"}, {"id"=>"2", "content"=>"This book is awesome"}, {"id"=>"3", "content"=>"I love this book"}], "author"=>{"id"=>"1", "name"=>"Drew Neil"}}, {"id"=>"2", "title"=>"Docker for Rails Developers", "comments"=>[{"id"=>"4", "content"=>"Awesome book"}, {"id"=>"5", "content"=>"Truly powerful"}, {"id"=>"6", "content"=>"Beautiful content"}], "author"=>{"id"=>"2", "name"=>"Rob Isenberg"}}]}} >
```

As you can see, we are calling 2 queries for comments and 2 queries for authors which obviously not efficient and would cause N+1 queries issue. To fix this, we would need include the followings inside `graphql_book_app_schema.rb`:

```rb
GraphqlBookAppSchema = GraphQL::Schema.define do
  # ...
  use GraphQL::Batch
  enable_preloading
end
```

Then change the definition of book as followings:

```rb
# app/graphql/types/book_type.rb

require 'graphql/batch'
Types::BookType = GraphQL::ObjectType.define do
  name 'Book'

  field :id, !types.ID
  field :title, !types.String

  field :author, Types::AuthorType do
    preload :author # additional call to preload author
    resolve lambda { |obj, _args, _ctx|
      obj.author
    }
  end

  field :comments, Types::Comment do
    preload :comments # additional call to preload comments
    resolve lambda { |obj, _args, _ctx|
      obj.comments
    }
  end

  field :number_of_comments, !types.Int do
    preload :comments # additional call to preload comments
    resolve lambda { |obj, _args, _ctx|
      obj.comments.length
    }
  end
end
```

And then try to call the query again:

```rb
result = GraphqlBookAppSchema.execute(query)
```

The number of queries will be reduced as we have combined the query of comments and authors into one for each

```
Book Load (0.3ms)  SELECT `books`.* FROM `books`
Comment Load (0.2ms)  SELECT `comments`.* FROM `comments` WHERE `comments`.`book_id` IN (1, 2)
Author Load (0.2ms)  SELECT `authors`.* FROM `authors` WHERE `authors`.`id` IN (1, 2)
```

Now, we need to create a mutation query for logged in user. First, let's create `json_web_token.rb` inside lib folder:

```rb
# lib/json_web_token.rb

class JsonWebToken
  class << self
    def encode(payload, exp = 24.hours.from_now)
      payload[:exp] = exp.to_i
      JWT.encode(
        payload,
        Rails.application.credentials.secret_key_base
      )
    end

    def decode(token)
      body = JWT.decode(
        token,
        Rails.application.credentials.secret_key_base
      )[0]
      HashWithIndifferentAccess.new body
    rescue
      nil
    end
  end
end
```

and require this file in `application.rb` like following:

```rb
# config/application.rb
# ...

require './lib/json_web_token.rb'
```

Then we need to define a GraphQL type called `LoggedInUser`:

```rb
# app/graphql/mutations/logged_in_user.rb

class Mutations::LoggedInUser < GraphQL::Function
  # define the arguments this field will receive
  argument :email, !Types::AuthInput

  # define what this field will return
  type Types::AuthType

  # resolve the field's response
  def call(_obj, args, _ctx)
    input = args[:email]
    return unless input

    user = User.find_by(email: args[:email])
    return unless user
    return unless user.authenticate(args[:password])

    OpenStruct.new(jwt: AuthToken.token(user),
                   user: user)
  end
end
```

and then create `auth_type.rb` like followings:

```rb
# app/graphql/types/auth_type.rb

require 'graphql/batch'
Types::AuthType = GraphQL::ObjectType.define do
  name 'AuthType'

  field :jwt, !types.String
  field :user, Types::UserType
end
```

and `auth_input.rb`:

```rb
# app/graphql/types/auth_input.rb

Types::AuthInput = GraphQL::InputObjectType.define do
  name 'AuthInput'

  argument :email, !types.String
  argument :password, !types.String
end
```

and create a model called `AuthToken`, which is responsible for generating the token and verifying if the input token is legit

```rb
# app/models/auth_token.rb

class AuthToken
  def self.token(user)
    payload = { user_id: user.id }
    JsonWebToken.encode(payload)
  end

  def self.verify(token)
    result = JsonWebToken.decode(token)
    return nil unless result
    User.find_by(id: result[:user_id])
  end
end
```

and declare this class in `mutation_type.rb` like followings:

```rb
# frozen_string_literal: true

Types::MutationType = GraphQL::ObjectType.define do
  name 'Mutation'

  field :logged_in_user, function: Mutations::LoggedInUser.new
end
```

We would also need to add in the current_user retrieval inside `graphql_controller.rb` like followings:

```rb
class GraphqlController < ApplicationController
  def execute
    # ...
    context = {
      current_user: current_user
    }
    # ...
  end

  private

  # ...

  def current_user
    return nil if request.headers['Authorization'].blank?
    token = request.headers['Authorization'].split(' ').last
    return nil if token.blank?
    AuthToken.verify(token)
  end
end
```

To verify we can call this logged in user query, open rails console and try to run the followings:

```rb
query = %(mutation {
    logged_in_user(
      auth: {
        email: "admin@example.com",
        password: "secret"
      }) {
      jwt,
      user {
        id
        email
      }
    }
  }
)
result = GraphqlBookAppSchema.execute(query)

# => #<GraphQL::Query::Result @query=... @to_h={"data"=>{"logged_in_user"=>{"jwt"=>"eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoyLCJleHAiOjE1MjU3MDc2MDl9.jYLl7P9yWSLOzUcSLEiEVwKjJwZCTkXj-5JwaE6sBaY", "user"=>{"id"=>"2", "email"=>"admin@example.com"}}}} >
```

To allow cross origin request (CORS), we would need to add in the following inside application.rb:

```rb
# ...
module GraphqlBookApp
  class Application < Rails::Application
    # ...
    config.middleware.use Rack::Cors do
      allow do
        origins '*'
        resource '*',
                 headers: :any, 
                 methods: %I[get post put delete options]
      end
    end
  end
end

# ...
```

To ensure that only logged in user can query books data, we would need to modify ``query_type.rb`` to check for current_user object in the context:

```rb
# app/graphql/types/query_type.rb

Types::QueryType = GraphQL::ObjectType.define do
  name 'Query'

  field :books, !types[Types::BookType] do
    resolve lambda { |_obj, _args, ctx|

      # added code to check for context of current_user
      if ctx[:current_user].blank?
        raise GraphQL::ExecutionError, 'Authentication required'
      end
      ##

      Book.all
    }
  end
end
```

We are ready on the Rails app side, let's move on to setting up the React app

## Setup React App

Let's start by create a React App using ``create-react-app`` by issuing the command:

```bash
create-react-app graphql_book_react
```

Inside this project folder, create a file call ``.env`` to set the development port:

```
# .env
PORT=3002
```

Inside ``App.js``, remove the existing sample code and replace with the folllowings:

```js
{% raw %}
import React, { Component } from 'react'
import logo from './logo.svg'
import './App.css'

class App extends Component {
  state = { email: 'admin@example.com', password: 'secret', jwt: '', books: [] }

  login() {
    # TODO
  }

  renderBooksListing() {
    let bookRecords = this.state.books.map(book => {
      return (
        <tr key={book.id}>
          <td>{book.id}</td>
          <td>{book.title}</td>
          <td>{book.author.name}</td>
          <td>{book.number_of_comments}</td>
        </tr>
      )
    })
    return (
      <table className="table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Title</th>
            <th>Author</th>
            <th>Comments Count</th>
          </tr>
        </thead>
        <tbody>{bookRecords}</tbody>
      </table>
    )
  }

  renderLogin() {
    return (
      <div>
        <h3>Login</h3>
        <form>
          <div className="form-group">
            <input
              type="email"
              placeholder="Email"
              className="form-control"
              value={this.state.email}
              onChange={e => this.setState({ email: e.target.value })}
            />
          </div>
          <div className="form-group">
            <input
              type="password"
              placeholder="Password"
              className="form-control"
              value={this.state.password}
              onChange={e => this.setState({ password: e.target.value })}
            />
          </div>
          <div className="form-group">
            <button className="btn btn-primary" onClick={e => this.login(e)}>
              Login
            </button>
          </div>
        </form>
      </div>
    )
  }

  render() {
    return (
      <div className="App">
        {this.state.jwt == '' && this.renderLogin()}
        {this.state.jwt != '' && this.renderBooksListing()}
      </div>
    )
  }
}

export default App

{% endraw %}
```

We have defined ``renderBooksListing`` and ``renderLogin`` functions. These are to render HTMLs for the list of books and the login form respectively. There are no logic of how we login or how we retrieve the books at the moment.

To speed up things, I am preseting the email & password to the login that we have but remember to remove these once you moved into production environment :D

I also put up the bootstrap css inside ``index.html`` like followings:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <!-- more code -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css" integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm" crossorigin="anonymous">   <link rel="shortcut icon" href="%PUBLIC_URL%/favicon.ico">
  </head>
  <!-- more code -->
</html>
```

If you launch the react app using the command: ``yarn start``, you should see the login form now. Hitting Login button button now will not do anything. Let's add in the code for login action:

```js
class App extends Component {
  login(e) {
    e.preventDefault()

    // we define a query here
    let query = `mutation {
      logged_in_user(
        auth: {
          email: "${this.state.email}"
          password: "${this.state.password}"
        }) {
        jwt,
        user {
          id
          email
        }
      }
    }`

    fetch('http://localhost:3000/graphql', {
      method: 'post',
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json'
      },
      body: JSON.stringify({
        query: query
      })
    })
      .then(response => {
        return response.json()
      })
      .then(data => {
        let user = data.data.logged_in_user
        // if a non nil logged_in_user object is returned
        if (user) {
          this.setState({ jwt: user.jwt }, () => {
            // call retrieve books once we have the jwt
            this.retrieveBooks()
          })
        } else {
          alert('Incorrect username or password')
        }
      })
  }

  // ... more code ...
}
```

The code is pretty much self explained. We define the query. Then we use fetch to call the API which would be hitting the graphql controller. After retrieving back the data, we check if the ``logged_in_user`` object is non-nil. If it is, we woudl call ``retrieve_books`` function. Otherwise, we would alert about the incorrectness of username & password.

We would also need to write ``retrieve_books`` function once we got the JWT from the logged in user:

```js
class App extends Component {

  // ... more code ...
  retrieveBooks() {
    let query = `query {
          books {
            id,
            title,
            number_of_comments,
            author {
              id,
              name
            }
          }
        }`

    fetch('http://localhost:3000/graphql', {
      method: 'post',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${this.state.jwt}`,
        Accept: 'application/json'
      },
      body: JSON.stringify({
        query: query
      })
    })
      .then(response => {
        return response.json()
      })
      .then(data => {
        this.setState({ books: data.data.books })
      })
  }
  // ... more code ...
}
```

Once we got the jwt from user, we would then again define the books query. Then calling fetch with the query and the headers ``Authorization`` with the jwt token. Note that, the API call will not be successful and throw back Authorization Required message if there is no jwt token supplied.

Once we get the data from the GraphQL API, we would then set the book records back into the state so that it could render out. 

You could launch the Rails server and run the React app at the same time and try it out to see the end result now. It's pretty awesome with GraphQL as now, we have a single API end point for almost everything that we need and API users can freely retrieve things that he/she needs to without altering the backend.

I have published the code here for your reference:

[https://github.com/jameshuynh/blog-codes/tree/master/graphql_book](https://github.com/jameshuynh/blog-codes/tree/master/graphql_book)
