---
layout: post
title:  "How does Docker work?"
date:   2019-25-03
---

How does Docker _actually_ work? It's a simple question that has a surprisingly complex answer. You've probably heard the words "daemon" and "runtime" thrown around in documentation and other blog posts, but never really understood what they were and how they fit together. If you're like me and went wading through the source code to uncover the truth, I can understand if you got lost in the sea of code. Let's face it, if Docker source code was a meal, you'd be chowing down on a huge bowl of spaghetti. 

Like a fork guiding pasta to your mouth, this post will group and guide the strands of Docker to your hungry mind.

In order to better understand the present, we first need to look at the past. In 2013 Solomon Hykes of [dotCloud](https://www.crunchbase.com/organization/dotcloud#section-overview) revealed Docker to the public at the PyCon talk [_The future of Linux Containers_](https://www.youtube.com/watch?v=wW9CAH9nSLs). Let's revert his [git respository](https://github.com/moby/moby/tree/bba4e368077cbc73db2a12c259c5fc2330dffe75) to January of 2013, to a simpler time in Docker's development.

> Note: the moby/moby and docker/docker-ce repo's share the same tree of commits at this point in time

# How did Docker work in 2013

DIAGRAM

OVERVIEW

### Command-line Application

The Docker command line application is your interface to control containers managed by Docker. It's relatively simple as all of the controlling is done by the _dockerd_ component. In the main function, it interprets commands from the user and sends them to the docker daemon. An HTTP Request is made to the address stored in the DOCKER environment variable (default value: SOMETHING).  

<https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/docker/docker.go>

When dockerd replies with results, the command line application will display these to the user.

### Daemon

The component known as the "daemon" ... PURPOSE

<https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go>

The docker daemon lives here, the main function will start an HTTP listener on port 8080, and a TCP listener on port 4242

<https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/tcp.go>

This will listen for requests and process them with a goroutine.

<https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/types.go>

Using reflection, we find out what method we need to call in order to process the request. We then call it!

### `docker run`

Let's call the run command

https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L591

If we did not provide an image, we juse use "base". If we did not specify a command, we just run "/bin/bash -i".

We find the image that was specified, whether by name or ID
https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/image/image.go#L94

and then we create the container.

https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L527

The code to create a container lives here:

https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L50

It does similar things to how linux containers are created. Makes a new root directory, various things about stdin and stdout.

The config for this container is saved in config.json in the root directory! (Hey we've seen this file before!)

we then create what's known as an LXCTemplate file, which we fill in with all the necessary variables that our container needs. The templated config file is here:
https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/lxc_template.go#L7

LXC is --- TODO


Then back to our run command, we attach stdin and stdout, then finally we run our container:

https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L651

Starting the container involves: 
https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L255

using lxc-start

in a seperate thread we then monitor the process in order to cleanup the contianer when it finishes; https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L334

We then do a synchronisation block to wait until the container has finished running:
https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L670
https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/state.go#L57

This only happens if you run a container in the foreground. If you want to run in the background, you just print the container ID and exit. 

# Whats changed?

how many lines of code changed?

So that was 2013, how does Docker work in 2018?

runtime?
shim?

Open Container Initiatve started in 2015

# How does it work now

The docker code base is, let's face it, a hot mess. But hopefully this paints a good enough picture to wrap your head around how it works.

THE CLI APP IS NOW OUT OF THE CODE BASE?

The client to communcate with the daemon has grown considerably, here is the function which instructs the daemon to start a container:
https://github.com/moby/moby/blob/master/client/container_start.go

This will send an HTTP post request to the docker daemon. https://github.com/moby/moby/blob/master/client/request.go

By default, the docker daemon will have the address /var/run/docker.sock
https://github.com/moby/moby/blob/master/client/client_unix.go

We can command the docker daemon through:
https://github.com/moby/moby/blob/master/cmd/dockerd/docker.go#L52

and trigger it to start running: https://github.com/moby/moby/blob/472a52861c93539222015e614cc041ac4b79a483/cmd/dockerd/daemon.go#L74

It will start an HTTP listener, waiting to process requests:
https://github.com/moby/moby/blob/472a52861c93539222015e614cc041ac4b79a483/cmd/dockerd/daemon.go#L588

The server begins its routing here:
https://github.com/moby/moby/blob/8e610b2b55bfd1bfa9436ab110d311f5e8a74dcb/api/server/router/container/container.go#L30

This function will receive the post request to start a container
https://github.com/moby/moby/blob/852542b3976754f62232f1fafca7fd35deeb1da3/api/server/router/container/container_routes.go#L170

which then will pass over to the backend to start a container:
https://github.com/moby/moby/blob/8e610b2b55bfd1bfa9436ab110d311f5e8a74dcb/daemon/start.go#L18

