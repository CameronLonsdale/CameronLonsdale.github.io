---
layout: post
title:  "How does Docker work?"
date:   2019-03-25
---

How does Docker _actually_ work? It's a simple question that has a surprisingly complex answer. You've probably heard the terms "daemon" and "runtime" thrown around in documentation and blog posts, but never really understood what they meant and how they fit together. If you're like me and went wading through the source to uncover the truth, you're not alone if you drowned in the sea of code. Let's face it, if Docker source code was a meal, you'd be chowing down on a big bowl of spaghetti.

Like a fork that guides pasta to your mouth, this post will group and guide the digital strands of Docker into your hungry mind.

In order to better understand the present, we first need to look at the past. In 2013 Solomon Hykes of [dotCloud](https://www.crunchbase.com/organization/dotcloud#section-overview) revealed Docker to the public at the PyCon talk [_The future of Linux Containers_](https://www.youtube.com/watch?v=wW9CAH9nSLs). Let's revert his [git repository](https://github.com/moby/moby/tree/bba4e368077cbc73db2a12c259c5fc2330dffe75) to January of 2013, to a simpler time in Docker's development.

<div class="padded-highlight">
    <pre>Note: the moby/moby and docker/docker-ce repo's share the same tree of commits at this point in time.</pre>
</div>

# How did Docker work in 2013?

TODO: FIX THIS IMAGE

<img src="{{ site.baseurl }}/assets/img/docker-work/architecture_2013.png">

Docker is composed of two main components, a command-line application for users and a daemon which tracks and manages containers. The daemon relies on two sub components to perform its job, storage on the host file system for image and container data; and the LXC interface to abstract away the raw kernel calls needed to construct a Linux container.

## Command-line Application

The Docker command-line application is the human interface to managing all images and containers known to your running copy of Docker. It's relatively simple since all of the management is done by the _dockerd_ component. The app starts at the [main function](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/docker/docker.go#L161):

{% highlight go %}
func main() {
    var err error
    ...
    // Example:              "/var/run/docker.sock", "run"
    conn, err := rcli.CallTCP(os.Getenv("DOCKER"), os.Args[1:]...)
    ...
    receive_stdout := future.Go(func() error {
        _, err := io.Copy(os.Stdout, conn)
        return err
    })
    ...
}
{% endhighlight %}

Immediately, a TCP connection is established to an address which is stored in the environment variable _DOCKER_, this is the address of the Docker daemon. The user supplied arguments are sent, and the app is now waiting to print out the results from a successful reply.

## dockerd

In the same repo lives the code for what's known as the docker daemon, or _dockerd_. Its job is to run in the background, cleaning up containers, and processing user requests. Upon [start-up](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L680) _dockerd_ will listen for incoming HTTP connections on port 8080, and TCP connections on port 4242.

{% highlight go %}
func main() {
    ...
    // d is the server, it will process requests made by the user
    d, err := New()
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

Once a command has been [received](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/tcp.go#L36), _dockerd_ will lookup the [function](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/tcp.go#L59) to be run using [reflection](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/types.go#L41), and then call it to perform the user's actions.

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

The user will normally provide an image and command for _dockerd_ to run. When they are omitted, the image <code class="inline-highlight">base</code> and command <code class="inline-highlight">/bin/bash -i</code> are used.

#### Find the image

Then we find the specified image by [mapping the name](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/image/image.go#L94) (or id) to a location on the file system (assuming an image already exists due to a previous <code class="inline-highlight">docker pull</code>).

{% highlight go %}
// Find the image
img := srv.images.Find(name)
if img == nil {
    return errors.New("No such image: " + name)
}
{% endhighlight %}

{% highlight go %}
type Index struct {
    Path    string // "/var/lib/docker/images"
    ByName  map[string]*History
    ById    map[string]*Image
}

func (index *Index) Find(idOrName string) *Image {
    ...
    // Lookup by ID
    if image, exists := index.ById[idOrName]; exists {
        return image
    }
    // Lookup by name
    if history, exists := index.ByName[idOrName]; exists && history.Len() > 0 {
        return (*history)[0]
    }
    return nil
}
{% endhighlight %}

In this version of Docker all images are stored in the folder [<code class="inline-highlight">/var/lib/docker/images</code>](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/dockerd/dockerd.go#L687). To learn more about what's in a docker image, see my [previous blog post](https://cameronlonsdale.com/2018/11/26/whats-in-a-docker-image/).

#### Create the container

Then we [create the container](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/docker.go#L49). _dockerd_ creates a structure to hold all the metadata related to this container, and stores it in a list for easy access.

{% highlight go %}
container := &Container{    // Examples
    Id:         id,         // "09906fa3"
    Root:       root,       // "/var/lib/docker/containers/09906fa3/"
    Created:    time.Now(),
    Path:       command,    // "/bin/bash -i"
    Args:       args,
    Config:     config,

    // "/var/lib/docker/containers/09906fa3/rootfs"
    // "/var/lib/docker/containers/09906fa3/rw"
    // TODO: WORK OUT WHETHER TO DO LAYERS OR NOT
    Filesystem: newFilesystem(path.Join(root, "rootfs"), path.Join(root, "rw"), layers),
    State:      newState(),

    // "/var/lib/docker/containers/09906fa3/config.lxc"
    lxcConfigPath: path.Join(root, "config.lxc"),
    stdout:        newWriteBroadcaster(),
    stderr:        newWriteBroadcaster(),
    stdoutLog:     new(bytes.Buffer),
    stderrLog:     new(bytes.Buffer),
}
{% endhighlight %}

When [creating the struct](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/container.go#L50) a [unique directory](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/container.go#L75) is made for the container at the path <code class="inline-highlight">/var/lib/docker/containers/&lt;ID&gt;</code>. Inside this path are [two directories](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/filesystem.go#L24), <code class="inline-highlight">/rootfs</code> which is the read-only root file system (the layers from the image that have been union mounted), and <code class="inline-highlight">/rw</code> to have a separate read-write layer for the container to create temporary files.

Last, an [LXC config file](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/container.go#L175) is generated by filling in a [template](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/lxc_template.go) with our newly created container data. More on LXC in the next section.

#### Run the container

Our container is finally created! But it's not yet running, for that we need to [start](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/container.go#L188) it.

{% highlight go %}
func (container *Container) Start() error {
    if err := container.Filesystem.EnsureMounted(); err != nil {
        return err
    }

    params := []string{
        "-n", container.Id,
        "-f", container.lxcConfigPath,
        "--",
        container.Path,
    }
    params = append(params, container.Args...)

    // /usr/bin/lxc-start -n 09906fa3 -f /var/lib/docker/containers/09906fa3/config.lxc -- "/bin/bash -i"
    container.cmd = exec.Command("/usr/bin/lxc-start", params...)
    ...
}
{% endhighlight %}

The first step is to make sure the container's file system is [mounted](https://github.com/moby/moby/blob/f8f9285ccaeb35a2d5909a03f48f9d3b9d34aca2/filesystem.go#L34).

{% highlight go %}
func (fs *Filesystem) Mount() error {
    ...
    rwBranch := fmt.Sprintf("%v=rw", fs.RWPath)
    roBranches := ""
    for _, layer := range fs.Layers {
        roBranches += fmt.Sprintf("%v=ro:", layer)
    }
    branches := fmt.Sprintf("br:%v:%v", rwBranch, roBranches)

    // Mount the branches onto "/var/lib/docker/containers/09906fa3/rootfs"
    if err := syscall.Mount("none", fs.RootFS, "aufs", 0, branches); err != nil {
        return err
    }
    ...
}
{% endhighlight %}

Using the [AUFS](https://en.wikipedia.org/wiki/Aufs) union mount file system, the layers of an image are mounted read-only on top of each other to present one coherent view to the container. The read-write path is mounted on the top layer to provide the container with temporary storage.

Then, to start the container, _dockerd_ runs another program _lxc-start_ with the LXC template we just generated.

#### LXC

[LXC](https://linuxcontainers.org/lxc/introduction/) (Linux Containers) is an abstraction layer which provides userspace applications with a simple API to create and manage containers. The truth is, containers are [not a real thing](https://blog.jessfraz.com/post/containers-zones-jails-vms/), there is no such object called a container inside the Linux Kernel. Containers are a collection of kernel objects that work together to provide process isolation. Therefore, the simple [<code class="inline-highlight">lxc-start</code>](https://linuxcontainers.org/lxc/manpages//man1/lxc-start.1.html#LXC) command actually translates into the setup and application of:

* Kernel namespaces (ipc, uts, mount, pid, network and user)
* Apparmor and SELinux profiles
* Seccomp policies
* Chroots (using pivot_root)
* Kernel capabilities
* and CGroups (control groups)

#### Cleanup

{% highlight go %}
func (container *Container) monitor() {
    // Wait for the program to exit
    container.cmd.Wait()
    exitCode := container.cmd.ProcessState.Sys().(syscall.WaitStatus).ExitStatus()

    // Cleanup
    container.stdout.Close()
    container.stderr.Close()
    if err := container.Filesystem.Umount(); err != nil {
        log.Printf("%v: Failed to umount filesystem: %v", container.Id, err)
    }

    // Report status back
    container.State.setStopped(exitCode)
    container.save()
}
{% endhighlight %}

Finally, _dockerd_ will then [monitor](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L334) the container til completion, cleaning up unnecessary data now that the container has finished.

# What's changed?

It's been 6 years since the introduction of Docker, and the containerisation paradigm has exploded in popularity.

OPEN CONTAINER INITIATIVE 2015 (because people wanted to create containers in other ways than just LXC??)

SOMETHING ABOUT THE SIZE OF THE DOCKER PROJECT

WINDOWS CONTAINERS

# How does Docker work in 2019?

After 6 years and 36,207 commits the moby repo has evolved into a large collaborative project, influencing and relying upon many components.

TODO ARCHITECTURE:

To better understand these architectural changes, let's compare moby 2013 to [moby 2019](https://github.com/moby/moby/tree/468eb93e5acc809248405102db32460fe7efed08).

## Command-line Application

The control flow of the command-line application for the most part hasn't changed. A request is made to the daemon, it waits for a reply and then print the results. Today, HTTP(S) with JSON encoded bodies is the standard for communicating with _dockerd_.

To allow for extensibility, the API and the docker binary were separated. The program code lives at [docker/cli](https://github.com/docker/cli), which relies upon the [moby/moby/client](https://github.com/moby/moby/tree/468eb93e5acc809248405102db32460fe7efed08/client) package for the interface to talk to _dockerd_.

## Dockerd

Dockerd will start [listening](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/cmd/dockerd/daemon.go#L584) for HTTP user requests, and process them according to [routes](https://github.com/moby/moby/tree/468eb93e5acc809248405102db32460fe7efed08/api/server/router) predefined by the API. NOT SURE IF PREDEFINED IS THE BEST WORD

{% highlight go %}
// Container Routes
func (r *containerRouter) initRoutes() {
    r.routes = []router.Route{
        // HEAD
        router.NewHeadRoute("/containers/{name:.*}/archive", r.headContainersArchive),
        // GET
        router.NewGetRoute("/containers/json", r.getContainersJSON),
        router.NewGetRoute("/containers/{name:.*}/export", r.getContainersExport),
        router.NewGetRoute("/containers/{name:.*}/changes", r.getContainersChanges),
        ...
    }
} 
{% endhighlight %}

The engine is still responsible for a variety of tasks, like interacting with [image registries](https://github.com/moby/moby/tree/master/distribution) and setting up directories on the file system for use by containers. TODO: FIND THIS!!!

An absent feature from the 2019 daemon is that it no longer manages the life cycle of running containers. As the project grew, the decision was made to split off container supervision into a separate project, _containerd_.

Although [docker/engine](https://github.com/docker/engine) is forked from moby/moby allowing possible code divergence, they share the same commit tree to date.

### docker run

{% highlight go %}
func runContainer(dockerCli command.Cli, opts *runOptions, copts *containerOptions, containerConfig *containerConfig) error {
    ...
    // create the container
    createResponse, err := createContainer(ctx, dockerCli, containerConfig, &opts.createOptions)
    ...
    // start the container
    client.ContainerStart(ctx, createResponse.ID, types.ContainerStartOptions{});
    ...
}
{% endhighlight %}

A [<code class="inline-highlight">docker run</code>](https://github.com/docker/cli/blob/afde31d710c2057960b35332f0be6c6c2aeaf3c9/cli/command/container/run.go#L95) command begins with a call to the daemon to create a container. This request is routed to [<code class="inline-highlight">postContainersCreate</code>](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/api/server/router/container/container_routes.go#L443).

A [couple](https://github.com/moby/moby/blob/a3eda72f71962cbe413795fcf496d63aa8f15a7a/daemon/create.go#L39) [function](https://github.com/moby/moby/blob/a3eda72f71962cbe413795fcf496d63aa8f15a7a/daemon/create.go#L55) [calls](https://github.com/moby/moby/blob/a3eda72f71962cbe413795fcf496d63aa8f15a7a/daemon/create.go#L103) later and we're creating a container.

{% highlight go %}
func (daemon *Daemon) create(opts createOpts) (retC *container.Container, retErr error) {
    var (
        container *container.Container
        img       *image.Image
        imgID     image.ID
        err       error
    )
    ...
    container = daemon.newContainer(opts.params.Name, os, opts.params.Config, opts.params.HostConfig, imgID, opts.managed);
    ...
    // Set RWLayer for container after mount labels have been set
    rwLayer = daemon.imageService.CreateLayer(container, setupInitLayer(daemon.idMapping))
    container.RWLayer = rwLayer
    ...
    idtools.MkdirAndChown(container.Root, 0700, rootIDs);
    idtools.MkdirAndChown(container.CheckpointDir(), 0700, rootIDs);

    // IS THIS IMPORTANT??
    if err := daemon.createContainerOSSpecificSettings(container, opts.params.Config, opts.params.HostConfig); err != nil {
        return nil, err
    }
    ...
    daemon.Register(container);
    ...
}
{% endhighlight %}

TODO KEEP GOING

Once the container has been created, we then make the call to start running it.

TODO

# Containerd

Containerd self describes as a "container runtime", however I find this misleading. Borrowing the description given by its author Michael Crosby, I would say Containerd is better thought of as a container supervisor. TODO MORE.

TODO something about runc?

Then it just passes a config and file system path to containerd????

Client handle from here
https://github.com/moby/moby/blob/a3eda72f71962cbe413795fcf496d63aa8f15a7a/libcontainerd/libcontainerd_linux.go

To create
https://github.com/moby/moby/blob/master/libcontainerd/remote/client.go#L210

and start
https://github.com/moby/moby/blob/master/libcontainerd/remote/client.go#L240

THEN:

On the server side:
Create
https://github.com/containerd/containerd/blob/1ac546b3c4a3331a9997427052d1cb9888a2f3ef/services/tasks/local.go#L112

Create the task
https://github.com/containerd/containerd/blob/35582cb7a33bd33b2693174acad5d303332671c0/runtime/v1/linux/runtime.go#L154


and 

Start task
https://github.com/containerd/containerd/blob/1ac546b3c4a3331a9997427052d1cb9888a2f3ef/services/tasks/local.go#L181

which involves creating this process
https://github.com/containerd/containerd/blob/1ac546b3c4a3331a9997427052d1cb9888a2f3ef/runtime/task.go#L35

On linux this is started here: https://github.com/containerd/containerd/blob/1ac546b3c4a3331a9997427052d1cb9888a2f3ef/runtime/linux/process.go#L124

Where it uses the "shim" of the task to start the process

https://github.com/containerd/containerd/blob/9ed2c0aa021948d405b54e1d6ea7fde63bd8cd60/runtime/v1/linux/proc/init_state.go#L83

https://github.com/containerd/containerd/blob/9ed2c0aa021948d405b54e1d6ea7fde63bd8cd60/runtime/v1/linux/proc/init.go#L258

Then off to runc

WHERE DOES THE CLEANUP MONITORING HAPPEN?

RunC
----
(what is also refered to as a runtime in the code, but no)
https://github.com/containerd/go-runc/blob/master/runc.go#L181

https://github.com/opencontainers/runc

Extra
-----
https://www.youtube.com/watch?v=VWuHWfEB6ro
https://containerd.io/img/architecture.png
https://www.youtube.com/watch?v=sK5i-N34im8
https://blog.docker.com/2016/04/docker-containerd-integration/
https://groups.google.com/forum/#!topic/docker-dev/zaZFlvIx1_k
https://www.youtube.com/watch?v=ZAhzoz2zJj8


TODO: Ask Aleksa to review your post!




https://github.com/containerd/containerd

https://github.com/containerd/containerd/blob/8f63d2acdbca7082164d63222c1efe010e01c5e3/client.go#L222

Create
https://github.com/containerd/containerd/blob/c09932fcb01009cefc49f8731d863f58954e3c74/containerstore.go#L109





Client then calls protobuf API to create
https://github.com/containerd/containerd/blob/master/api/services/containers/v1/containers.pb.go#L421

WHERE IS THE SERVICE RUNNING? WHATS THE ADDRESS??

Implementation of the service:
https://github.com/containerd/containerd/blob/155d7acb014671bc675367ec99cad8144548802c/services/containers/service.go

https://github.com/containerd/containerd/blob/155d7acb014671bc675367ec99cad8144548802c/services/containers/local.go#L107

https://github.com/containerd/containerd/blob/d5f00ed9138ca88bd4d693c89b9633b84e11efc9/metadata/containers.go#L106


then talks to the shim???
https://github.com/containerd/containerd/blob/06e04bc5a9e35dcd471cb5e20d0ca20b28fae730/runtime/v1/shim/service.go#L116

https://github.com/containerd/containerd/blob/06e04bc5a9e35dcd471cb5e20d0ca20b28fae730/runtime/v1/shim/service.go#L635



