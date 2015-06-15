---
layout: site
body_class: background-primary
title: Puma + Sidekiq + Heroku
description: Configuring a puma server with sideqik in heroku
---

In a recent project I've been working we decided to go with a [Puma](http://puma.io/) + [Sidekiq](http://sidekiq.org/)
configuration deployed on Heroku. Even though heroku has a great documentation on setting up puma,
when you introduce the factor of sidekiq there are a some tweaks that you need to do in order to prevent connections errors,
be it with redis or postgres. Here are some of the things we had to do for our app to work without problems.

All this configurations are with Rails 4.1+ in mind.

### Puma

Setting up Puma is pretty straight forward following [heroku's recommendation](https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-serve):

{% highlight ruby linenos %}
# app/config/puma.rb
workers Integer(ENV['WEB_CONCURRENCY'] || 2)
threads_count = Integer(ENV['MAX_THREADS'] || 3)
threads threads_count, threads_count

preload_app!

rackup      DefaultRackup
port        ENV['PORT']     || 3000
environment ENV['RACK_ENV'] || 'development'

on_worker_boot do
  # Worker specific setup for Rails 4.1+
  # See: https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server#on-worker-boot
  ActiveRecord::Base.establish_connection
end
{% endhighlight %}

### Sidekiq

The mayor limitation when using sidekiq on heroku is the connection limit imposed by redis to go. For the nano plan they offer 10 connections and this is where most of your calculations must focus.

Thankfully, Manuel van Rijn built a great [tool](http://manuel.manuelles.nl/sidekiq-heroku-redis-calc/) to help you with the calculations.
Basically you must take into account the number of connections to redis from your web process,
the number of sidekiq threads from you workers and add two reserved connections from the sidekiq server itself (per worker).
The tool is built with unicorn in mind but can be applied for puma, just think of the Web threads param as the web concurrency of your puma server.

This leaves our sidekiq configuration as follows:

{% highlight yaml linenos %}
# app/config/sidekiq.yml
:concurrency: <%= ENV['SIDEKIQ_THREADS'] || 4 %>
:queues:
  - default
{% endhighlight %}

{% highlight ruby linenos %}
# app/config/initializers/sidekiq.rb
Sidekiq.configure_client do |config|
  config.redis = { size: 1 }
end

Sidekiq.configure_server do |config|
  config.redis = { size: ENV['SIDEKIQ_REDIS'] || 6 }
end
{% endhighlight %}

### Database
We had database connection problems with the pool size limit being reached when we set the pool size to the exact number of threads in sidekiq,
even adding one or two extra connections gave us problems, so we decided to work with some extra db connections since our plan allowed us and on top of that use one of the new features of rails to reap idle connections.
We set the rails app to kill any idle connection after 5 seconds, to do so you have to add the reaping_frequency property to your database.yml and a value that represent the number of seconds to wait for a connection to be terminated.

{% highlight yaml linenos %}
# app/config/database.yml
pool: <%= ENV['DB_POOL'] || ENV['MAX_THREADS'] || 3 %>
reaping_frequency: <%= ENV['REAPING_FREQUENCY'] || nil %>
{% endhighlight %}

The reaping_frequency is not set by default since apparently there are some issues with MySQL, so if it is your case be warned and check the status of this issues.

### Wrapping up

I hope you can use this guide to fine tune your server when you decide to work with sidekiq and heroku.
It was a lot of reading different sources, recommendations, and try/error to get our application running smooth.

We are currently working on two web dynos on heroku, using a standard Postgres plan and a nano Redis To Go plan.
Our current environment variables for server concurrency and Sidekiq are as follow:

{% highlight bash linenos %}
# ENV variables
DB_POOL=10
REAPING_FREQUENCY=5 # Set the number of seconds for rails to terminate idle db connections
WEB_CONCURRENCY=2 # Puma process
MAX_THREADS=3 # Puma threads
SIDEKIQ_THREADS=4 # Sidekiq threads
SIDEKIQ_REDIS=6 # Sidekiq redis connections
{% endhighlight %}

And our Procfile:

{% highlight bash linenos %}
# Procfile
web: bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq
{% endhighlight %}

### References
- [https://blog.codeship.com/puma-vs-unicorn/](https://blog.codeship.com/puma-vs-unicorn/)
- [https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-serve](https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-serve)
- [http://manuel.manuelles.nl/sidekiq-heroku-redis-calc/](http://manuel.manuelles.nl/sidekiq-heroku-redis-calc/)
- [http://manuel.manuelles.nl/blog/2012/11/13/sidekiq-on-heroku-with-redistogo-nano/](http://manuel.manuelles.nl/blog/2012/11/13/sidekiq-on-heroku-with-redistogo-nano/)
