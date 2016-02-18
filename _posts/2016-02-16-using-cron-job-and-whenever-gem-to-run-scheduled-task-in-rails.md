---
layout: post
title: Using cron job and whenever gem to run scheduled task in Rails
comments: true
excerpt_separator: <!-- more -->
---

Scheduled task is quite a common task in Rails. It can be scheduling to send a reminder email, or scheduling to crawl something from another website on daily basic. Rails does not have support for this type of feature out of the box, but we can use the combination between cron job and whenever gem to achieve the purpose. In this article, let's quickly go through how to use this combination.

<!-- more -->

## Cron

Cron is a time-based job scheduler in Unix-like computer operating systems. It can be used to schedule jobs to run periodically at fixed times, dates, or intervals. Cron is most suitable for scheduling repetitive tasks. (definition taken from [The Wikipedia](https://en.wikipedia.org/wiki/Cron))

In a linux server, you can open cron job file using the following command:

{% highlight bash %}
crontab -e
{% endhighlight %}

Each of a cron job is specified using the following command syntax (taken from [drupal.org](https://www.drupal.org/node/23714)):

{% highlight bash %}
# +---------------- minute (0 - 59)
# |  +------------- hour (0 - 23)
# |  |  +---------- day of month (1 - 31)
# |  |  |  +------- month (1 - 12)
# |  |  |  |  +---- day of week (0 - 6) (Sunday=0)
# |  |  |  |  |
  *  *  *  *  *  command to be executed
{% endhighlight %}

There are __5__ variables that can be changed inside a cron task:

- 1st variable: Minute, which can be specified from __0 to 59__
- 2nd variable: Hour, which can be specified from __0 to 23__
- 3rd variable: Day of Month, which can be specified from __1 to 23__
- 4th variable: Month, which can be specifed from __1 to 12__
- 5th variable: Day of Week, which can be specifed from __0 - 6__ (Sunday = 0)

If the variable is a __*__ it means for all
if the variable is a series of number split by comma (,), it means OR condition on each of the specified number.

{% highlight bash %}
0,30  *  *  *  *  command to be executed
{% endhighlight %}

This job would run on __0th minute__ OR __30th minute__ in all hours, all days, all  months and all days of week, which would be equivalent to running this job every 30 minutes starting from minute 0

Similarly, if we a command to run every 10 minutes starting from minute 0, we can specify as below

{% highlight bash %}
0,10,20,30,40,50  *  *  *  *  command to be executed
{% endhighlight %}

The same thing applies for hour, day of month, month and day of week. For instance,

{% highlight bash %}
0,30  *  *  1,2,3 0,1,2  command to be executed
{% endhighlight %}

This would mean run this job on every 30 minutes starting from minute 0 only on Jan, Feb and March (e.g. 1,2,3 string) and only on Sunday, Monday and Tuesday (e.g. 0,1,2 string)

As you can see, although it is quite straight forward, remembering the position of each of the variables and the values each variables can take could be quite challenging. Besides, analyzing the requirement of when to run a task is not obvious, sometimes you would hear something as like the following:

- Run a task every Monday at 9am
- Run a task every 3 months at mid night
- Run a task every 5 minutes

Although, we can convert the above requirements to a approriate cron jobs, it might take sometimes. That's why __Whenever__ gem has come to help to speed up the solving of this problem by giving us a tool to generate the cron job line.

## Whenever Gem

Whenever Gem is a Gem to help generating human readable syntax to cron job syntax and at the same time help to put the job into cron specification file. You can go through the README of whenver in the following github URL:

[https://github.com/javan/whenever](https://github.com/javan/whenever)

### 1. Installation

Put this line inside your ``Gemfile``:

{% highlight ruby %}
gem 'whenever', require: false
{% endhighlight %}

Then run

{% highlight bash %}
bundle install
{% endhighlight %}

### 2. Usage

Initialise whenverize by running this command in your Rails folder:

{% highlight bash %}
wheneverize .
{% endhighlight %}

This command will generate the file ``config/schedule.rb``. You can then add tasks into this ``config/schedule.rb`` file.

### 3. Core Features

Following are its core feature that are pretty useul for our daily work:

#### 3.1 Ability to convert human readable cron syntax to the original cron job syntax of linux

This is my most favorite feature, as now we can write a task in ``config/schedule.rb`` something like below:

{% highlight ruby %}
every :monday, at: '9am' do
  rake "some_rake_task"
end
{% endhighlight %}

Whenever will then be able to convert to:

{% highlight bash %}
0 9 * * 1 /bin/bash -l -c 'cd /Users/james/apps/sample_app && RAILS_ENV=production bundle exec rake some_rake_task --silent'
{% endhighlight %}

By looking at the line ``0 9 * * 1``, it is quite daunting to understand. But it is so much easier by looking at the line ``every :monday, at: '9am'``

Another example:

{% highlight ruby %}
every 3.months, at: '0am' do
  rake "some_rake_task"
end
{% endhighlight %}

will translate to

{% highlight bash %}
0 0 1 1,4,7,10 * /bin/bash -l -c 'cd /Users/james/apps/sample_app && RAILS_ENV=production bundle exec rake some_rake_task --silent'
{% endhighlight %}

#### 3.2 Tighly integrate with capistrano

In ``config/deploy.rb``, if you add in the following line in the require section:

{% highlight ruby %}
require 'whenever/capistrano'
{% endhighlight %}

whenever would translate the tasks in ``schedule.rb`` and put in cron job file automatically every time you deploy the app. Neat!

## Some useful tips on cron

### 1. Log out the output from a cron job

It is important to see the log output from a cron job in order to determine if anything goes wrong during runtime. The default task generated by whenever does not log the output to anywhere, which could be difficult to debug and understand what has gone wrong. In order to counter that, I would usually manually copy the output from whenever and add in the ``>`` portion to output logs to a file

In order to display the cron jobs generated by whenever, you can simply go to Rails folder on the server. Mine is usally the ``current`` folder and run the command:

{% highlight bash %}
bundle exec whenever
{% endhighlight %}

This would output all the jobs in ``config/schedule.rb`` in cron job syntax, hence you can copy them to your cron specification file manually. I would then add in the following

{% highlight bash %}
/home/app/some_rake_task_cron.log 2>&1
{% endhighlight %}

You can of course change the log file path to fit your preference. After this my task would look like following:

{% highlight bash %}
0 2 * * * /bin/bash -l -c 'cd /home/app/www/sample_app/current && RAILS_ENV=production bundle exec rake some_rake_task --silent' > /home/app/some_rake_task_cron.log 2>&1
{% endhighlight %}

### 2. Change dynamic release folder in whenever task to current folder

A generated cron job would be something like following

{% highlight bash %}
0 1 * * * /bin/bash -l -c 'cd /home/app/www/sample_app/releases/20160218095451 && RAILS_ENV=production bundle exec rake some_rake_task --silent'
{% endhighlight %}

which contains the releases folder. However, this release folder would change every time you deploy. If you choose to copy the task manually to cron and you are afraid of forgeting to change the cron job everytime you deploy, remember to change the output command to use current folder instead of release folder:

{% highlight bash %}
0 1 * * * /bin/bash -l -c 'cd /home/app/www/sample_app/current && RAILS_ENV=production bundle exec rake some_rake_task --silent'
{% endhighlight %}

### 3. Change bundle exec rake to use bin/rake instead

If you use rbenv or rvm on the server, ``bundle exec rake`` will not always run. On Rails 4 onwards, ``rake`` is actually generated inside ``bin`` folder in Rails root. Hence you can utilise that in your cron job by changing ``bundle exec rake`` in your whenever output cron commands to ``bin/rake``. Below is my final output cron task that is ready to be copied over to cron specification file:

{% highlight bash %}
0 1 * * * /bin/bash -l -c 'cd /home/app/www/sample_app/releases/20160218095451 && RAILS_ENV=production bin/rake some_rake_task --silent' > /home/app/some_rake_task_cron.log 2>&1

{% endhighlight %}


