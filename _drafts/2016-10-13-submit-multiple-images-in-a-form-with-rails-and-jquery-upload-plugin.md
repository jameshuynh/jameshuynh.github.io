---
layout: post
title: Submit multiple images in a form with Rails and jQuery File Upload Plugin
date: 2016-10-13
comments: true
categories: [rails, blueimp, upload]
tags: [rails, blueimp, upload]
excerpt_separator: <!-- more -->
---

jQuery File Upload (or Blueimp) is an extremly useful plugin that can help to submit multiple files in an HTML form. However, I feel the documents are not that friendly. It's hard to find the right resources to do certain things. For instance, recently my colleague asked me how to submit multiple images in a Rails form together with its data and I have spent couple of hours looking through the documents to find out what are the right things to do even before I can start doing the development. That's the motivation for this article.

In this article, I would like to show how to implement a sample Rails form with multiple images upload that can run on IE10 and above and on other browsers.

<!-- more -->

## New Rails project and setup

Let begin by generating a new Rails project. If you already have an existing Rails project, you can skip this step:

```bash
rails new blueimp-images-upload --database=mysql
cd blueimp-images-upload
bundle exec rake db:create
```

For the purpose for the demo, we are going to create a form which can create a Book and upload multiple cover images for a book. Now, let's create a book entity

```bash
rails g model Book title description:text
```

Then we will create a book cover entity

```bash
rails g model BookCover book_id:integer
```

Next, add ``paperclip`` gem into your ``Gemfile``

```bash
# Gemfile

gem 'paperclip', '~> 5.1.0'
```

And bundle install:

```bash
bundle install
```

Then generate a paperclip attachment for ``BookCover``

```bash
rails g paperclip BookCover photo
```

And migrate all the tables:

```bash
bundle exec rake db:migrate
```

Next, we would need to generate a books controller with 4 views which are ``index``, ``new``, ``edit`` and ``show``

```bash
rails g controller books index new edit show
```

Now, let's open the project and add in some Ruby code to define the relationship and ``accepts_nested_attributes_for`` in order to let us submit the relationship through a form

```ruby
# book.rb
class Book < ApplicationRecord
  has_many :book_covers, dependent: :destroy
  accepts_nested_attributes_for :book_covers, allow_destroy: true
end
```

And declare the paperclip attachment and its validation

```ruby
# book_cover.rb
class BookCover < ApplicationRecord
  has_attached_file :photo, styles: { thumb: "220x310>" }
  validates_attachment_content_type :photo, content_type: /\Aimage\/.*\Z/
end
```

We also need to add in ``routes.rb`` the ``resources`` route for book

```ruby
# routes.rb
Rails.application.routes.draw do
  resources :books
end
```

## Views and Form

We are done with the initial setup, now let's create some views and the form:

First would be ``index`` view

```erb
<!-- books/index.html.erb -->
```

Next, we would create a partial ``_form.html.erb`` to contain the form. This form would be shared between ``edit`` and ``new`` view

```erb
<!-- books/_form.html.erb -->
```

Next, we would then render form into ``new`` and ``edit`` view

```erb
<!-- books/new.html.erb -->
```

```erb
<!-- books/edit.html.erb -->
```

Finally, let create ``show`` view to show off a book with multiple cover images

```erb
<!-- books/show.html.erb -->
```

## Conclusion
