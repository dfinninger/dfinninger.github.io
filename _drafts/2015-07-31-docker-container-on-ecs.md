---
layout: post
title:  "Deploying Rails to AWS ECS"
date:   2015-07-31 10:54:45
categories:
---

If you have a container with an application inside, you need to be able to deploy that container to somewhere so that others can experience your wonderful application. Sometimes, engineers will spin up VMs to run Docker containers... but then we are left with a lot of the same issues we had with VMs _and_ we get the added complexity of Docker running on top of that.

Amazon has released the Amazon Web Services EC2 Container Service (AWS ECS) to run containers without the fuss of VMs.

In this post we'll take a look at how to get an app running on the service.

## What I'm Using
---

I have a rails app that I'd like to deploy onto ECS. It is already in a container and on the Docker Hub.

If you do not have this, please take a look at my previous post here: [Deploying Rails With Docker]({% post_url 2015-07-31-deplying-rails-with-docker %})
