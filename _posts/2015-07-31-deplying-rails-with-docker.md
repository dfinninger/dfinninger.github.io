---
layout: post
title:  "Deploying Rails with Docker"
date:   2015-07-31 08:09:23
categories:
---

I've been asked by a few people recently how to deploy a rails app into docker. So this will be a write up of a few different components and approaches.

_A few things we'll cover:_

- Getting a Rails app to the "Welcome aboard!" page.
- Looking at the phusion-passenger docker container
- Implementing a dockerfile to build out the app
- Pushing the application to Docker Hub

So, without further ado...

## Welcome aboard! (Pre-Reqs)
---

### Quick list of things I'm using:

- RVM
- Ruby 2.1.6
- Rails 4.2.3
- Amazon Linux (for what we are doing CentOS should be a drop-in replacement)
  - The only place that this really matters is on how to install docker

### How I set it up:

[You can install RVM here.][RVM-install]

From there we need to install a ruby and install rails. The RVM install page lists instructions for doing this in one step, but I like seperating it out a bit.

This will get us ruby: `rvm install ruby-2.1.6`

This will get us rails: `rvm gemset create rails && gem install rails -N`

And then generate an app in the directory of your choice: `rails new hello_world`

_Note: There are far better posts about installing/setting up rails, so you should be able to flush out any errors with a quick google._

Throw up a `rails server` to make sure things are working.

## Getting Docker
---

Here are the commands to install on a Yum-based system. If you are not using one, please refer to the documentation: [here][docker-docs].

{% highlight bash %}
sudo yum update -y
sudo yum install -y docker
sudo service docker start
{% endhighlight %}

Now enable docker for your username: `sudo usermod -a -G docker USERNAME`

And we should be able to use docker without sudo: `docker info`

If you don't have a Docker Hub account, get one now: [here][docker-signup]. Remember your user/pass.

_Note: If you are on Mac or Windows, you'll be using boot2docker. I'm not going to cover that here, but the docs (see above) are pretty solid and we aren't doing anything too complicated with Docker. So, in the words of Douglas Adams: "DON'T PANIC"_

## Passenger Container and Dockerfile
---

Phusion have made a pretty slick webserver that we'll be using. But they have also made one hell of a docker container. Their README is amazing. Everything you need to know is in it. Because of it's size, we'll be taking a quick tour of the features needed to run a rails app. There is so much more in this container that I highly recommend looking over.

The way to customize this container is to build our own using this one as a base. We can do that with a Dockerfile.

In the repository that houses our application code, move up a directory and put a Dockerfile there. See this tree for reference:

{% highlight bash %}
.
└── appdir
    ├── Dockerfile
    └── hello_world
        ├── app
        ├── bin
        ├── config
        ... etc ...
{% endhighlight %}

The first line of our Dockerfile should be the `FROM` statement. This will tell docker which image to pull. Then there are a few things recommended in the README that we should set.

{% highlight bash %}
# Dockerfile
# Note, modify the version (0.9.16) to the newest one
FROM phusion/passenger-ruby21:0.9.16

# Set correct environment variables.
ENV HOME /root

# Use baseimage-docker's init process.
CMD ["/sbin/my_init"]

# =====================
#  Our build goes here
# =====================
{% endhighlight %}

Now let's set up the webserver. First, we need to give Nginx a default site to locate the app. Let's use this one Phusion gives us:

{% highlight bash %}
# /etc/nginx/sites-enabled/hello_world.conf:
server {
    listen 80;
    server_name www.example.com;
    root /home/app/hello_world/public;

    passenger_enabled on;
    passenger_user app;

    passenger_ruby /usr/bin/ruby2.1;
}
{% endhighlight %}

Put that next to the Dockerfile in your machine. That way, the build process can pull in the file and put it into the container.

Note that `passenger_user` is set to "app". This is because this container has a special user for running our app.

Now, for the next section of the Docekrfile:

{% highlight bash %}
# ... another section of Dockerfile

# Enable Nginx
RUN rm -f /etc/service/nginx/down

# Set up our Nginx site
RUN rm /etc/nginx/sites-enabled/default
ADD hello_world.conf /etc/nginx/sites-enabled/hello_world.conf
{% endhighlight %}

And for this simple Rails App, we can insert it into the container with a few lines in the Dockerfile:

{% highlight bash %}
COPY hello_world /home/app/hello_world
RUN gem install bundler -N
RUN cd /home/app/hello_world && bundle install`
{% endhighlight %}


### Relevant Files

Here's the full files for your reference/ctrl-c:

Dockerfile:

{% highlight bash %}
# Dockerfile
# Note, modify the version (0.9.16) to the newest one
FROM phusion/passenger-ruby21:0.9.16

# Set correct environment variables.
ENV HOME /root

# Use baseimage-docker's init process.
CMD ["/sbin/my_init"]

# Enable Nginx
RUN rm -f /etc/service/nginx/down

# Set up our Nginx site
RUN rm /etc/nginx/sites-enabled/default
ADD hello_world.conf /etc/nginx/sites-enabled/hello_world.conf

# Push app into container
COPY hello_world /home/app/hello_world

# Get Bundler
RUN gem install bundler -N
RUN cd /home/app/hello_world && bundle install
{% endhighlight %}

Nginx conf:

{% highlight bash %}
# /etc/nginx/sites-enabled/hello_world.conf:
server {
    listen 80;
    server_name www.example.com;
    root /home/app/hello_world/public;

    passenger_enabled on;
    passenger_user app;
    passenger_app_env development;

    passenger_ruby /usr/bin/ruby2.1;
}
{% endhighlight %}

### Building the container

Make sure that you are in the same directory as the Dockerfile. We are going to use the `-t` flag to specify a tag.

Tags are of the form "user/repository:version". So my command looks like this - yours will differ. Make sure that the user is the username you chose for the Docker Hub earlier.

`docker build -t dfinninger/hello-world .`

This will build up the Docker container. When it is complete, run the generated container.

`docker run -dP dfinninger/hello-world`

The `-d` flag daemonizes the contianer and the `-P` flag auto-connects ports from the host to the container.

Now we need to know the correct port. Here's what I get:

{% highlight bash %}
[dfinninger@laptop appdir]$ docker ps
CONTAINER ID        IMAGE                           COMMAND             CREATED             STATUS              PORTS                                           NAMES
5a4cada4f997        dfinninger/hello-world:latest   "/sbin/my_init"     15 minutes ago      Up 15 minutes       0.0.0.0:32777->80/tcp, 0.0.0.0:32776->443/tcp   determined_blackwell
{% endhighlight %}

What we are interested in is the `PORTS` section. Looking at this: `0.0.0.0:32777->80/tcp` we can see that the host port 32777 has been connected to the guest port 80. So, if we point a browser to "http://localhost:32777" we will be routed to port 80 inside the container. So navigating to that address brings me to the "Welcome Aboard" page for rails.

![Welcome Aboard]({{ site.url }}/assets/welcome_aboard.png)

Sweet, Rails is now in a container.

## Docker Hub
---

The Docker Hub is a massive collection of containers. We can upload our own into it distribute it from there.

Log into the Docker Hub online and go to your profile. Click the big "Create Repository" button. We need to create a repo that is the same as our tag on the container we built. Mine is "dfinninger/hello-world", so I need to create a repository named "hello-world".

_Note: Make sure that the repo is set to "public". It makes things easier for this tutorial._

Once the repo is created we can push to it.

Back on your machine run `docker login`. This will authenticate with the hub and give you write access to your namespace.

Now that I am logged in, I can push up my image with `docker push dfinninger/hello-world`. This can take a few minutes as all of your image layers push to the Hub.

When that is complete you can pull your container down to anywhere and run it with the same command we used before!

## Wrap-Up
---

Let's quickly summarize the steps we took to get here:

1. Installed our environment and generated the rails skeleton.
1. Installed Docker
1. Created a Dockerfile
1. Looked into the phusion/passenger-ruby21 container and configured it according to our needs
1. Built our container with the app inside
1. Pushed our container to the Docker Hub

[RVM-install]: https://rvm.io/rvm/install
[docker-docs]: https://docs.docker.com/installation/
[docker-signup]: https://hub.docker.com/account/signup/
