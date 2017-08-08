---
layout: post
title: Restore remote MySQL Database into your local development in Rails
date: 2017-08-08
comments: true
categories: [ruby, mysql, database, rails]
tags: [ruby, mysql, database, rails]
excerpt_separator: <!-- more -->
---

Since the day I started deploying to a remote server until recently, I have always had to load database from staging server or production server into my local Rails development environment for some debugging purposes. However, it is not a simple process and usually occupy some of my times with quite a bit of manual commands.

Recently, I have decided this have to be changed and that moment I have started writing a Ruby gem just to do this mundate task.

<!-- more -->

I wrote a Gem called ``load_remote_db`` and it is live in [https://rubygems.org/gems/load_remote_db](https://rubygems.org/gems/load_remote_db). You can install this gem by adding the line into ``development`` group like below:

```ruby
# ...
group :development do
  gem 'load_remote_db'
end
# ...
```

And then run bundle install:

```bash
bundle install
```

You will need ``capistrano`` version 2 or ``mina`` tasks in place to have this run. To trigger to database restore from ``staging`` server into your local development environment, simply issue the command:

```bash
bundle exec rake db:load_from_remote
```

This will default load the staging database dump into your local development environment. If you want to change to load the production database dump into your local development environment, simply add ``SERVER=production`` into the command:

```bash
bundle exec rake db:load_from_remote SERVER=production
```

### Download database dump only

If you are only interested in download the database dump, you can add in ``DOWNLOAD_ONLY=true``:

```bash
bundle exec rake db:load_from_remote DOWNLOAD_ONLY=true
```

This will simply download SQL file into your local Rails root folder and trigger the open command to open Finder (on Mac OSX)

### Download user uploaded folder

If you want to download user uploaded folder like ``public/system`` (created by ``paperclip``) or ``public/uploads`` (created by ``carrierwave``), you can add in the option ``SYNC_FOLDER=public/system`` or ``SYNC_FOLDER=public/uploads``.


```bash
bundle exec rake db:load_from_remote SYNC_FOLDER=public/system
```

With this ``load_remote_db`` gem, I am now much faster swapping the database around between my local development environment, staging or production database to debug the issues only happen on these environments. I hope that you will find this useful too.
