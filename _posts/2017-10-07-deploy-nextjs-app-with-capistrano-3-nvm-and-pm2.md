---
layout: post
title: Deploy NextJS app with Capistrano 3, NVM and PM2 on Ubuntu server
date: 07-10-2017
comments: true
categories: [nextjs, react, capistrano, nvm, pm2]
tags: [nextjs, react, capistrano, nvm, pm2]
excerpt_separator: <!-- more -->
---

I was recently tasked to deploy a NextJS app into a ubuntu server. The requirement was that the deployment needs to be fast and as reliable as possible. Although I have been using mina for some of the apps, mina still does not really support multiple server instances deployment, which is quite a disappointment. Luckily, capistrano 3 supports multiple server deployment out of the box and pretty fast, reliable to be chosen as a deployment tool for all our projects. In this article, I am going to go through how I use Capistrano 3, Node Version Manager (NVM) and PM2 (a Node JS process manager) to deploy reliably a NextJS project.

<!-- more -->

## 1. Add Gemfile to NextJS project

We already have a NextJS project setup and the first thing to do now is create a ``Gemfile`` at the root project folder with the following content:

```ruby
# Gemfile
source 'https://rubygems.org'

gem 'capistrano', '~> 3.9.1'
```

After that you can run bundle install command to install the capistrano gems and capistrano Gem

```bash
bundle install
```

## 2. Add pm2-next task to package.json

Add the ``pm2-next`` line inside your ``scripts`` part in ``package.json`` file, like below. We will revisit this later on when we need to run this task:

```js
{
  "scripts": {
    "dev": "next",
    "build": "next build",
    "start": "next start",
    "pm2-next": "next build && pm2 start app.json"
  },
}
```

## 3. Setup capistrano

To setup capistrano on the current project, run this command:

```bash
bundle exec cap install
```

This will create a few files that we will need to edit later on:

```bash
config/deploy.rb
config/deploy/production.rb
Capfile
```

You can see that it also creates ``config/deploy/staging.rb`` but let's skip it for now as we can simply duplicate the ``config/deploy/production.rb`` and change the IP addresses and the domain name of the server.

## 4. Modify ``Capfile``

You can change your ``Capfile`` to have the following code. Note that, I have cleaned up all the commented code in the original file:

```ruby
# Capfile
require 'capistrano/setup'
require 'capistrano/deploy'
require 'capistrano/scm/git'
require 'capistrano/nvm'
install_plugin Capistrano::SCM::Git
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

## 5. Modify ``config/deploy/production.rb``

This is required as we would need to indicate what are the IP addresses of the servers that we would like to deploy the app to and the folder path located on the servers. You can simply copy the following code inside and make a modification from there:


```ruby
set :stage, :production
set :branch, 'master'

set :full_app_name, "#{fetch(:application)}_#{fetch(:stage)}"
set :server_name, 'nextjs-deployment.jameshuynh.com' # change to your application domain name

server 'xxx.xxx.xxx.xxx', user: 'ubuntu', roles: 'app', primary: true # change to your server IP and your username

set :deploy_to, "/home/#{fetch(:deploy_user)}/www/#{fetch(:full_app_name)}"
```

These are things that you would need to change in the above code are:

- ``set :server_name, 'nextjs-deployment.jameshuynh.com'``: You would need to change the domain name of the app that you are intending to deploy (instead of using ``nextjs-deployment.jameshuynh.com``).
- ``server 'xxx.xxx.xxx.xxx', user: 'ubuntu', roles: 'app', primary: true``: You would need to change the server IP (instead of using ``xxx.xxx.xxx.xxx``) and the approriate user (instead of ``ubuntu``).

## 6. Modify ``config/deploy.rb``

We would need to modify ``config/deploy.rb`` to add additional tasks to indicate what things that need to start/restart during the deployment:


```ruby
require 'capistrano/nvm'
set :application, 'nextjs-deployment' # change to your app name
set :deploy_user, 'ubuntu' # change to your server user
set :keep_releases, 5
set :repo_url, 'git@github.com:jameshuynh/nextjs-deployment.git' # change to your git address

# for NVM
set :nvm_type, :user
set :nvm_node, 'v8.6.0' # change to your node version number
set :nvm_map_bins, %w[node npm yarn pm2 next]
set :nvm_custom_path, "/home/#{fetch(:deploy_user)}/.nvm/versions/node"
set :default_env,
    'PATH' => "/home/#{fetch(:deploy_user)}/.nvm/versions/node/v8.6.0/bin:$PATH"
set :nvm_path, "/home/#{fetch(:deploy_user)}/.nvm"

# share node_modules folder
set :linked_dirs, %w[node_modules]

# rubocop:disable BlockLength
# pm2 tasks
namespace :pm2 do
  task :start do
    on roles(:app) do
      within current_path do
        execute :npm, 'run build'
      end
      within current_path do
        execute :pm2, "start #{shared_path}/app.json"
      end
    end
  end

  task :restart do
    on roles(:app) do
      within current_path do
        execute :npm, 'run build'
      end
      within shared_path do
        execute :pm2, 'reload app.json'
      end
    end
  end

  task :stop do
    on roles(:app) do
      within current_path do
        execute :pm2, 'stop app.json'
      end
    end
  end
end

namespace :deploy do
  after 'deploy:publishing', 'deploy:yarn_install'
  after 'deploy:publishing', 'deploy:restart'

  task :initial do
    on roles(:app) do
      before 'deploy:restart', 'pm2:start'
      invoke 'deploy'
    end
  end

  task :yarn_install do
    on roles(:app) do
      within current_path do
        execute :yarn, 'install'
      end
    end
  end

  task :restart do
    invoke 'pm2:restart'
  end

  task :start do
    invoke 'pm2:start'
  end

  task :stop do
    invoke 'pm2:stop'
  end
end
```

These are things that you would need to change in the above code are:

- ``set :application, 'nextjs_deployment'``: You would need to change to the app name that you are deploying (instead of using ``nextjs_deployment``).
- ``set :deploy_user, 'ubuntu'``: You would need to change to the username of your servers.
- ``set :repo_url, 'git@github.com:jameshuynh/nextj-deployment.git'``: You would need to change the git URL where your app resides.
- ``set :nvm_node, 'v8.6.0'``: You would need to change to the node js version that you intend to use.

## 6. Install NVM on server

From server, simply trigger the following command to install NVM. Note that the nvm version at the time I wrote this article is ``0.33.5``. It might subject to change in the future:

```bash
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.5/install.sh | bash
```

After the installation has completed, run the following command to load nvm into the shell:


```bash
. ~/.bashrc
```

Finally, to complete this step, run the following command to install node 8.6.0 (current at the time I wrote this article)

```bash
nvm install 8.6.0
```

After the installation completes, set the nodejs by issuing the command:

```bash
nvm use 8.6.0
```

Now verify you have the correct nodejs version by issuing this command:

```bash
node --version
```

We would also need to install ``yarn`` and ``pm2`` as global for this NodeJS version by issuing the following commands:

```bash
npm -g install yarn
npm -g install pm2
```

## 7. Copy server id_rsa.pub key to your git repo deploy key

If you do not have an ``id_rsa.pub`` key, you can generate one by issuing the following command on the server:

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

And following the prompt in the command.

After that, the ``id_rsa.pub`` key should be located at:

```bash
~/.ssh/id_rsa.pub
```

You can simply copy the content of this file and set it as one of deploy keys in your git project setting. This would ensure that the server has the approriate right to pull the code from your git server.

## 7. Run capistrano check

Now, you could trigger the following command to ensure everything has setup accordingly:

```bash
bundle exec cap production deploy:check
```

If you following closely from step 1 to 6, capistrano should check off all the requirements that it need to do.

## 8. Prepare the app.json file inside shared folder

From the server, you would need to deposit a file called ``app.json`` into the project shared folder. This file would then be managed by PM2. The content of the ``app.json`` is as following:

```js
{
  "apps" : [{
    "name": "nextjs-deployment",
    "script": "/home/ubuntu/www/nextjs-deployment_production/current/node_modules/next/dist/bin/next-start",
    "cwd": "/home/ubuntu/www/nextjs-deployment_production/current",
    "exec_interpreter": "~/.nvm/versions/node/v8.6.0/bin/node",
    "instances": "max",
    "exec_mode": "cluster",
    "watch": true,
    "env": {
      "NODE_ENV": "production"
    }
  }]
}
```

You would need to change the followings to fit with your app:

- ``name``: You need to change it to your app name.
- ``script``: You will need to change ``/home/ubuntu/www/nextjs-deployment_production`` with the path that you are want the app to be deplyed to. It must be the same as the value of ``deploy_to`` that has been set in ``config/production.rb`` file
- ``cwd``: You will also need to change ``/home/ubuntu/www/nextjs-deployment_production`` with the path that you are want the app to be deplyed to. It must be the same as the value of ``deploy_to`` that has been set in ``config/production.rb`` file

## 9. Setup nginx to forward the request to 127.0.0.1:3000

Create an nginx config file in ``/etc/nginx/sites-available/nextjs-deployment.jameshuynh.com`` like the following. However, you would need to change the file name according to the domain name of the app that you are deploying to:

```
upstream nextjs_deployment { # change to your project name
  server 127.0.0.1:3000;
}

server {
  listen 80;
  client_max_body_size 1G;

  large_client_header_buffers 8 16k;
  server_name nextjs-deployment.jameshuynh.com; # change to your domain name

  root /home/ubuntu/www/nextjs-deployment_production/current; # change to the folder path that set in capistrano deploy_to

  try_files $uri/index.html $uri @nextjs_deployment; # change to your project name

  location @nextjs_deployment { # change to your project name
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    if (!-f $request_filename) {
      proxy_pass http://nextjs_deployment; # change to your project name
      break;
    }
 }

  keepalive_timeout 60;
  error_page 500 502 503 504 /500.html;

  access_log  /var/log/nginx/nextjs_deployment.log; # change to your project name
  error_log  /var/log/nginx/nextjs_deployment.log; # change to your project name
}
```

These are things that you would need to change in the above code are:

- ``upstream nextjs_deployment``: You would need to change to name of your project.
- ``server_name nextjs-deployment.jameshuynh.com;``: You would need to change to the domain name that you are deploying to.
- ``root /home/ubuntu/www/nextjs-deployment_production/current``: You would need to swap the part of ``/home/ubuntu/www/nextjs-deployment_production`` with the same path value that has been set as ``deploy_to`` in capistrano ``config/production.rb``
- ``try_files $uri/index.html $uri @nextjs_deployment;``: You would need to change ``nextjs_deployment`` with your project name.
- ``proxy_pass http://nextjs_deployment; ``: You would need to change ``nextjs_deployment`` with your project name.
- ``access_log  /var/log/nginx/nextjs_deployment.log;``: You would need to change ``nextjs_deployment`` with your project name.
- ``error_log  /var/log/nginx/nextjs_deployment.log;``: You would need to change ``nextjs_deployment`` with your project name.

After you have changed all the parameters, you would need to symlink the nginx config file into ``sites-enabled`` folder like following. Note that, you would need to change the command below with the correct path according to what is the file name that you have named earlier.

```bash
sudo ln -nfs /etc/nginx/sites-available/nextjs-deployment.jameshuynh.com /etc/nginx/sites-enabled/nextjs-deployment.jameshuynh.com
```

And then check and restart nginx

```bash
sudo nginx -t
# make sure nginx returns ok here

sudo service nginx restart
```

Then try to visit your site from browser to see if the page comes out correctly.

## 10. Make pm2 auto start at start up

From the server, simply trigger the command below to get the command that you can run to make pm2 auto start at start up:

```bash
pm2 startup
```

For my case, it came out with the following:

```
[PM2] Init System found: systemd
[PM2] To setup the Startup Script, copy/paste the following command:
sudo env PATH=$PATH:/home/ubuntu/.nvm/versions/node/v8.6.0/bin /home/ubuntu/.nvm/versions/node/v8.6.0/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

I just then need to copy the third line of the above which is:

```bash
sudo env PATH=$PATH:/home/ubuntu/.nvm/versions/node/v8.6.0/bin /home/ubuntu/.nvm/versions/node/v8.6.0/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

and run it. PM2 would then take care of the auto restart after the server boot up. I rebooted the server after this and PM2 got auto started right after the server boot up. Sweet.

## 11. Add monit to help to monitor pm2

From the server, you can install monit by running the following command:

```bash
sudo apt-get install monit
```

After that modify the file to enable to monit on the server ``/etc/monit/monitrc``. Search for the lines below and uncomment them:

```bash
set httpd port 2812 and
    use address localhost  # only accept connection from localhost
    allow localhost        # allow localhost to connect to the server and
```

Also, I need to comment the following line in this file as it is causing some weird issue:

```bash
# include /etc/monit/conf-enabled/*
```

 Then restart monit by running the command:

 ```bash
 sudo service monit restart
 ```

Then create a file called ``pm2.monit`` inside ``/etc/monit/conf.d/`` folder with the content below:

```bash
check process pm2
  with pidfile /home/ubuntu/.pm2/pm2.pid
  start program = "/bin/systemctl start pm2-ubuntu.service"
  stop program = "/bin/systemctl stop pm2-ubuntu.service"
```

Then restart monit so that it can take effect:

```bash
sudo service monit restart
```

That's it about how to deploy a NextJS app by using capistrano 3, NVM and PM2 on a ubuntu server. I hope you can follow and enjoy the article. The source code of this is published on the following github repository for your reference

[https://github.com/jameshuynh/nextjs-deployment](https://github.com/jameshuynh/nextjs-deployment)
