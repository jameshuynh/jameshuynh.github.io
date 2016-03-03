---
layout: post
title: Integrate with IBM Softlayer Object Storage
date: 2016-03-02
comments: true
categories: [rails, storage, cloud]
tags: [rails, storage, cloud, softlayer]
excerpt_separator: <!-- more -->
---

IBM SoftLayer Object Storage is a service exactly like Amazon Simple Storage Service (Amazon S3). It provides storage layer so that we can persist our file securely on the cloud. I have a chance to work with IBM Softlayer Object Storage during integration with our Rails app. IBM Softlayer Object Storage is relatively new and there are little materials regarding how to do this integration. In this article, I will wlkthrough how we use IBM Softlayer Object Storage to store and retrieve file by making use of Rails and Carrierwave gem.

<!-- more -->

## 1. New Rails app, Carrierwave and Fog gem

Let's start by creating a new Rails app by running the following command:

{% highlight bash %}
rails new ibm-softlayer-object-storage-demo
{% endhighlight %}

After rails project has been created, add the following gems to your Gemfile:

{% highlight ruby %}
gem 'carrierwave', '~> 0.10.0'
gem 'fog', '~> 1.37.0'
gem 'fog-softlayer', git: 'https://github.com/fog/fog-softlayer.git'
gem "mini_magick", '~> 4.4.0'
{% endhighlight %}

Run bundle install to install these new gems into the project:

{% highlight bash %}
bundle install
{% endhighlight %}

## 2. Configure Carrierwave to use IBM Softlayer object storage

We would need to create a a file ``initializers/carrierwave.rb`` to indicate a few parameters for the IBM Softlayer object storage credentials. From ``initializers`` folder inside your Rails app, create ``carrierwave.rb`` file with the following content:

{% highlight ruby %}
# initializers/carrierwave.rb

CarrierWave.configure do |config|
  config.storage = :fog
  config.fog_credentials = {
    provider: :softlayer,
    softlayer_username: ENV['SOFTLAYER_USERNAME'],
    softlayer_api_key: ENV['SOFTLAYER_API_KEY'],
    softlayer_cluster: ENV['SOFTLAYER_CLUSTER']
  }
  config.fog_directory = ENV['SOFTLAYER_CONTAINER']
end
{% endhighlight %}

As you can see, I am using ``ENV`` variable to protect the credentials. These ``SOFTLAYER_USERNAME``, ``SOFTLAYER_API_KEY``, ``SOFTLAYER_CLUSTER`` and ``SOFTLAYER_CONTAINER`` key in ``ENV`` are populated by my ``rbenv-vars``. You would need to install ``rbenv-vars`` plugin for your ``rbenv`` here:

[https://github.com/rbenv/rbenv-vars](https://github.com/rbenv/rbenv-vars)

and then create a file call ``.rbenv-vars`` in this Rails folder with the following content. __Note__: Replace the parameter with your real value:

{% highlight bash %}
SOFTLAYER_USERNAME=<username>
SOFTLAYER_API_KEY=<api_key>
SOFTLAYER_CLUSTER=<cluster>
SOFTLAYER_CONTAINER=<container>
{% endhighlight %}

You must also need to add ``.rbenv-vars`` to your ``.gitignore`` to prevent this file for being committed, pushed and exposed to the world.

- ``username``: is supposed to be your login username of IBM Softlayer
- ``api_key``: is API Key (Password) when you click on the link as indicated in the screenshot below.
- ``cluster``: is your cluster code as indicated in the cluster listing screenshot below.
- ``container``: like Amazon S3, it is the bucket name (IBM Softlayer use the terminology Container).

<p style='text-align:center;' markdown='1'><img src='/public/images/cluster_code.png' alt="Obtain Cluster Code" style='display:inline;'/></p>


<p style='text-align:center;' markdown='1'><img src='/public/images/credentials_link.png' alt="Credentials Link" style='display:inline;'/></p>

## 3. Prepare Avatar Uploader for Carrierwave

First, generate an uploader for Carrierwave by issuing the following command:

{% highlight bash %}
rails g uploader Avatar
{% endhighlight %}

Then replace the content of that file by the following:

{% highlight ruby %}
# uploaders/avatar_uploader.rb

# encoding: utf-8

class AvatarUploader < CarrierWave::Uploader::Base

  include CarrierWave::MiniMagick

  # Indicate storage to use Fog
  storage :fog

  # Resize to thumb if file is an image
  version :thumb, if: :image? do
    process resize_to_fill: [128, 128]
  end

  def store_dir
    "uploads/#{model.class.to_s.underscore}/#{mounted_as}/#{model.id}"
  end

  # Expiring URL to retrieve file from object storage server
  def expiring_url(expiring_time=60)
    storage = Fog::Storage.new(self.fog_credentials)
    file = storage.directories \
                  .get(self.fog_directory) \
                  .files.get(self.path)
    return file.url(Time.now + expiring_time)
  end

  protected

  def image?(file)
    file.content_type.start_with? 'image'
  end

end
{% endhighlight %}

## 4. Create a Rails model to use Avatar Uploader

Let's create a user model by issuing the following command:

{% highlight bash %}
rails g model User avatar
{% endhighlight %}

Then add the following line to ``user.rb`` file to use the avatar uploader:

{% highlight ruby %}
# user.rb

mount_uploader :avatar, AvatarUploader
{% endhighlight %}

Then run the migration from your terminal:

{% highlight bash %}
bundle exec rake db:migrate
{% endhighlight %}

## 5. Test uploading & retrieving user avatar

First, open rails console by issuing the following command from your terminal:

{% highlight ruby %}
bundle exec rails console
{% endhighlight %}

Trigger the following commands in console to see if it works:

{% highlight ruby %}
user = User.new
# => #<User id: nil, avatar: nil, created_at: nil, updated_at: nil>

user.avatar = File.open("/absolute/path/to/an/image")
# => #<File:/Users/james/apps/blog-codes/ibm-softlayer-object-storage-demo/public/avatars/avatar.jpg>

user.save
#   (0.2ms)  begin transaction
#  SQL (0.3ms)  INSERT INTO "users" ("avatar", "created_at", "updated_at") VALUES (?, ?, ?)  [["avatar", "avatar.jpg"], ["created_at", 2016-03-03 00:13:01 UTC], ["updated_at", 2016-03-03 00:13:01 UTC]]
#   (1.6ms)  commit transaction
=> true

puts user.avatar.expiring_url
# => "https://....objectstorage.softlayer.net:443/v1/.../uploads/user/avatar/1/avatar.jpg?temp_url_sig=...&temp_url_expires=1456964053"

puts user.avatar.thumb.expiring_url
# => "https://....objectstorage.softlayer.net:443/v1/.../uploads/user/avatar/1/thumb_avatar.jpg?...&temp_url_expires=1456964081"
{% endhighlight %}

As you can see, we can now store files and retrieve back both the thumbnail and the original URL. The URL is an expiring URL.

Finally, as usual, the source code of this article is made available on this github URL - [https://github.com/jameshuynh/blog-codes/tree/master/ibm-softlayer-object-storage-demo](https://github.com/jameshuynh/blog-codes/tree/master/ibm-softlayer-object-storage-demo). Enjoy and have fun integrating with IBM Softlayer Storage
