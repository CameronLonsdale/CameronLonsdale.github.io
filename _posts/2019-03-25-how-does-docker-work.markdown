---
layout: post
title:  "How does Docker work?"
date:   2019-03-25
---

How does Docker _actually_ work? It's a simple question that has a surprisingly complex answer. You've probably heard the terms "daemon" and "runtime" thrown around in documentation and blog posts, but never really understood what they meant and how they fit together. If you're like me and went wading through the source to uncover the truth, you're not alone if you drowned in the sea of code. Let's face it, if Docker source code was a meal, you'd be chowing down on a big bowl of spaghetti.

Like a fork that guides pasta to your mouth, this post will group and guide the digital strands of Docker into your hungry mind.

In order to better understand the present, we first need to look at the past. In 2013 Solomon Hykes of [dotCloud](https://www.crunchbase.com/organization/dotcloud#section-overview) revealed Docker to the public at the PyCon talk [_The future of Linux Containers_](https://www.youtube.com/watch?v=wW9CAH9nSLs). Let's revert his [git respository](https://github.com/moby/moby/tree/bba4e368077cbc73db2a12c259c5fc2330dffe75) to January of 2013, to a simpler time in Docker's development.

<div class="padded-highlight">
    <pre>Note: the moby/moby and docker/docker-ce repo's share the same tree of commits at this point in time</pre>
</div>

# How did Docker work in 2013

DIAGRAM

OVERVIEW

### Command-line Application

The Docker command-line application is the human interface to managing all images and containers known to Docker. It's relatively simple since all of the management is done by the _dockerd_ component. The app starts at the [main function](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/docker/docker.go#L161):

{% highlight go %}
func main() {
    var err error
    ...
    conn, err := rcli.CallTCP(os.Getenv("DOCKER"), os.Args[1:]...)
    ...
    receive_stdout := future.Go(func() error {
        _, err := io.Copy(os.Stdout, conn)
        return err
    })
    ...
}
{% endhighlight %}

Immediately, a TCP connection is established to an address which is stored in the environment variable _DOCKER_, this is the address of the Docker daemon. The user supplied arguments are sent, and the app is now waiting to print out the results from a succesful reply.

### Dockerd

In the same repo lives the code for what's known as the docker daemon, or _dockerd_. Its job is to run in the background, listening for user's commands. Upon [start-up](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L680) _dockerd_ will listen for incoming HTTP connections on port 8080, and TCP connections on port 4242.

{% highlight go %}
func main() {
    ...
    go func() {
        if err := rcli.ListenAndServeHTTP(":8080", d); err != nil {
            log.Fatal(err)
        }
    }()
    if err := rcli.ListenAndServeTCP(":4242", d); err != nil {
        log.Fatal(err)
    }
}
{% endhighlight %}

Once a command has been [recieved](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/tcp.go#L36), _dockerd_ will lookup the [function](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/tcp.go#L59) to be run using [reflection](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/types.go#L41), and then call it to perform the user's actions.

### docker run

One such function is [<code class="inline-highlight">CmdRun</code>](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L591), which corresponds to the <code class="inline-highlight">docker run</code> command.

{% highlight go %}
func (srv *Server) CmdRun(stdin io.ReadCloser, stdout io.Writer, args ...string) error {
    flags := rcli.Subcmd(stdout, "run", "[OPTIONS] IMAGE COMMAND [ARG...]", "Run a command in a new container")
    ...
    // Choose a default image if needed
    if name == "" {
        name = "base"
    }
    // Choose a default command if needed
    if len(cmd) == 0 {
        *fl_stdin = true
        *fl_tty = true
        *fl_attach = true
        cmd = []string{"/bin/bash", "-i"}
    }
    ...
}
{% endhighlight %}

The user will normally provide an image and command for dockerd to run. When they are omitted, the image "base" and command "/bin/bash -i" are used.

Then we find the specified image by [mapping the name](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/image/image.go#L94) (or id) to a location on the file system. In this version of docker all images are stored in the folder [/var/lib/docker/images](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/dockerd/dockerd.go#L687). To learn more about what's in a docker image, see my [previous blog post](https://cameronlonsdale.com/2018/11/26/whats-in-a-docker-image/).

{% highlight go %}
// Find the image
img := srv.images.Find(name)
if img == nil {
    return errors.New("No such image: " + name)
}
{% endhighlight %}

Then we [create the container](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/docker.go#L49). _dockerd_ creates a structure to hold all the metadata related to this container, then stores it in a list for easy access.

TODO NEED MORE EMBEDDED CODE HERE, DON"T WANT TO KEEP SWAPPING WINDOWS

When [creating the struct](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/container.go#L50) a [unique directory](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/container.go#L75) is created for the container at the path <code class="inline-highlight">/var/lib/docker/containers/&lt;ID&gt;</code> Inside this path are [two directories](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/filesystem.go#L24) <code class="inline-highlight">/rootfs</code> to point to the files from the image that are now inside the running container, and <code class="inline-highlight">/rw</code> to have a seperate read/write layer for the container to create temporary files.

Last, an [LXC config file](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/container.go#L175) is generated by filling in a [template](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/lxc_template.go) with our newly created container data. More on LXC in the next section.

Our container is finally created! But it's not yet running, for that we need to [start](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/container.go#L188) it.


{% highlight go %}
func (container *Container) Start() error {
    ...
    container.cmd = exec.Command("/usr/bin/lxc-start", params...)
    ...
}
{% endhighlight %}

TODO

Starting the container involves: 
https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L255

using lxc-start

in a seperate thread we then monitor the process in order to cleanup the contianer when it finishes; https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L334

We then do a synchronisation block to wait until the container has finished running:
https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L670
https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/state.go#L57

This only happens if you run a container in the foreground. If you want to run in the background, you just print the container ID and exit. 

# What's changed?

how many lines of code changed?

So that was 2013, how does Docker work in 2018?

runtime?
shim?

Open Container Initiatve started in 2015 because people wanted to create containers in other ways than just LXC

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

