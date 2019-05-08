---
layout: post
title:  "[REVIEW COPY] How does Docker work?"
date:   2019-03-25
---

How does Docker _actually_ work? It's a simple question that has a surprisingly complex answer. You've probably heard the terms "daemon" and "runtime" thrown around, but never really understood what they meant and how they fit together. If you're like me and went wading through the source to uncover the truth, you're not alone if you drowned in the sea of code. Let's face it, if Docker source code was a meal, you'd be chowing down on a big bowl of spaghetti.

Like a fork that guides pasta to your mouth, this post will group and guide the digital strands of Docker into your hungry mind.

In order to better understand the present, we first need to look at the past. In 2013 Solomon Hykes of [dotCloud](https://www.crunchbase.com/organization/dotcloud#section-overview) revealed Docker to the public at the PyCon talk [_The future of Linux Containers_](https://www.youtube.com/watch?v=wW9CAH9nSLs). Let's revert his [git repository](https://github.com/moby/moby/tree/bba4e368077cbc73db2a12c259c5fc2330dffe75) to January of 2013, to a simpler time in Docker's development.

# How did Docker work in 2013?

<img src="{{ site.baseurl }}/assets/img/docker-work/architecture_2013.svg" style="max-width: 500px; display: block; margin-left: auto; margin-right: auto;">

Docker is composed of two main components, a command-line application for users and a daemon which manages containers. The daemon relies on two sub components to perform its job, storage on the host file system for image and container data; and the LXC interface to abstract away the raw kernel calls needed to construct a Linux container.

## Command-line Application

The Docker command-line application is the human interface to managing all images and containers known to your running copy of Docker. It's relatively simple since all of the management is done by the daemon. The app starts at the [main function](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/docker/docker.go#L161):

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

Immediately, a TCP connection is established to an address which is stored in the environment variable _DOCKER_, this is the address of the Docker daemon. The user supplied arguments are sent, and the app is now waiting to print the results from a successful reply.

## dockerd

In the same repo lives the code for the docker daemon, known as _dockerd_. Its job is to run in the background processing user requests and cleaning up containers. Upon [start-up](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L680) _dockerd_ will listen for incoming HTTP connections on port 8080, and TCP connections on port 4242.

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

Once a command has been [received](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/tcp.go#L36), _dockerd_ will lookup and call the [function](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/tcp.go#L59) to be run using [reflection](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/rcli/types.go#L41).

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
    // Find the image
    img := srv.images.Find(name)
    ...
    // Create Container
    container := srv.CreateContainer(img, *fl_tty, *fl_stdin, *fl_comment, cmd[0], cmd[1:]...)
    ...
    // Start Container
    container.Start()
    ...
}
{% endhighlight %}

The user will normally provide an image and command for _dockerd_ to run. When they are omitted, the image <code class="inline-highlight">base</code> and command <code class="inline-highlight">/bin/bash</code> are used.

#### Find the image

Then we find the specified image by [mapping the name](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/image/image.go#L94) (or id) to a location on the file system (assuming an image already exists due to a previous <code class="inline-highlight">docker pull</code>).

{% highlight go %}
type Index struct {
    Path    string // "/var/lib/docker/images/index.json"
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

In this version of Docker all images are stored in the folder [<code class="inline-highlight">/var/lib/docker/images</code>](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/dockerd/dockerd.go#L698). To learn more about what's in a docker image, see my [previous blog post](https://cameronlonsdale.com/2018/11/26/whats-in-a-docker-image/).

#### Create the container

Then we [create the container](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/docker.go#L49). _dockerd_ creates a structure to hold all the metadata related to this container, and stores it in a list for easy access.

{% highlight go %}
container := &Container{    // Examples
    Id:         id,         // "09906fa3"
    Root:       root,       // /var/lib/docker/containers/09906fa3/"
    Created:    time.Now(),
    Path:       command,    // "/bin/bash"
    Args:       args,       // ["-i"]
    Config:     config,

    // "/var/lib/docker/containers/09906fa3/rootfs"
    // "/var/lib/docker/containers/09906fa3/rw"
    Filesystem: newFilesystem(path.Join(root, "rootfs"), path.Join(root, "rw"), layers),
    State:      newState(),

    // "/var/lib/docker/containers/09906fa3/config.lxc"
    lxcConfigPath: path.Join(root, "config.lxc"),
    stdout:        newWriteBroadcaster(),
    stderr:        newWriteBroadcaster(),
    stdoutLog:     new(bytes.Buffer),
    stderrLog:     new(bytes.Buffer),
}
...
// Create directories
os.Mkdir(root, 0700);
container.Filesystem.createMountPoints();
...
// Generate LXC Config file
container.generateLXCConfig();

{% endhighlight %}

When [creating the struct](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L50) a [unique directory](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L75) is made for the container at the path <code class="inline-highlight">/var/lib/docker/containers/&lt;ID&gt;</code>. Inside this path are [two directories](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/filesystem.go#L24), <code class="inline-highlight">/rootfs</code> which is the read-only root file system (the layers from the image that have been union mounted), and <code class="inline-highlight">/rw</code> to have a separate read-write layer for the container to create temporary files.

Last, an [LXC config file](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L175) is generated by filling in a [template](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/lxc_template.go) with our newly created container data. More on LXC in the next section.

#### Run the container

Our container is finally created! But it's not yet running, for that we need to [start](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L255) it.

{% highlight go %}
func (container *Container) Start() error {
    // Mount file system if not mounted
    container.Filesystem.EnsureMounted();

    params := []string{
        "-n", container.Id,
        "-f", container.lxcConfigPath,
        "--",
        container.Path,
    }
    params = append(params, container.Args...)

    // /usr/bin/lxc-start -n 09906fa3 -f /var/lib/docker/containers/09906fa3/config.lxc -- /bin/bash -i
    container.cmd = exec.Command("/usr/bin/lxc-start", params...)
    ...
}
{% endhighlight %}

The first step is to make sure the container's file system is [mounted](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/filesystem.go#L34).

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
    syscall.Mount("none", fs.RootFS, "aufs", 0, branches);
    ...
}
{% endhighlight %}

Using the [AUFS](https://en.wikipedia.org/wiki/Aufs) union mount file system, the layers of an image are mounted read-only on top of each other to present one coherent view to the container. The read-write path is mounted as the topmost layer to provide the container with temporary storage.

Then, to start the container, _dockerd_ runs another program _lxc-start_ with the LXC template we just generated.

#### LXC

[LXC](https://linuxcontainers.org/lxc/introduction/) (Linux Containers) is an abstraction layer which provides userspace applications with a simple API to create and manage containers. The truth is, containers are [not a real thing](https://blog.jessfraz.com/post/containers-zones-jails-vms/), there is no such object called a container inside the Linux kernel. Containers are a collection of kernel objects that work together to provide process isolation. Therefore, the simple [<code class="inline-highlight">lxc-start</code>](https://linuxcontainers.org/lxc/manpages//man1/lxc-start.1.html#LXC) command actually translates into the setup and application of:

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
    container.Filesystem.Umount();

    // Report status back
    container.State.setStopped(exitCode)
    container.save()
}
{% endhighlight %}

Finally, _dockerd_ will then [monitor](https://github.com/moby/moby/blob/bba4e368077cbc73db2a12c259c5fc2330dffe75/container.go#L334) the container til completion, cleaning up unnecessary data now that the container has finished.


## Summary

In summary, launching a container using Docker 2013 involves the following steps:

{% highlight text %}
dockerd is sent the run command (using the command-line application or otherwise)
    ↳ dockerd finds the specified image on the file system
    ↳ A container struct is created and stored for future use
    ↳ Directories on the file system are setup for use by the container
    ↳ LXC is instructed to start the container
    ↳ dockerd monitors the container until completion
{% endhighlight %}

# What's changed?

It's been 6 years since the introduction of Docker, and the containerisation paradigm has exploded in popularity. Both small and large enterprises have adopted Docker, especially in tandem with the orchestration system Kubernetes.

3 contributors turned into over 1800 through the power of Open Source, each person bringing with them new ideas for the project. Eager to promote extensibility, the [Open Container Initiative](https://www.opencontainers.org/) (OCI) was formed in 2015 to define an open standard around container formats and runtimes. The [image spec](https://github.com/opencontainers/image-spec) outlines the structure of a container image, and the [runtime spec](https://github.com/opencontainers/runtime-spec) describes the interface and behaviour that implementations should adhere to in order to run containers on their platform. As a result, the community developed a wide range of projects for container management, from native containers to ones isolated by a virtual machine. With support from Microsoft, the industry now has OCI compliant native Windows containers as well.

All of these changes have been reflected in the moby repo. With this historical context, we can begin deconstructing the components of Docker 2019.

# How does Docker work in 2019?

After 6 years and 36,207 commits the moby repo has evolved into a large collaborative project, influencing and relying upon many components.

<img src="{{ site.baseurl }}/assets/img/docker-work/architecture_2019.svg" style="max-width: 500px; display: block; margin-left: auto; margin-right: auto;">

In a very simplistic view, [Moby 2019](https://github.com/moby/moby/tree/468eb93e5acc809248405102db32460fe7efed08) has two new main components, _containerd_ which supervises containers during their lifetime, and OCI compliant runtimes (e.g _runc_) that are the lowest user level abstraction for creating containers (replacing LXC).

## Command-line Application

The control flow of the command-line application for the most part hasn't changed. Today, HTTP(S) with a JSON body is the standard for communicating with _dockerd_.

To allow for extensibility, the API and the docker binary were separated. The program code lives at [docker/cli](https://github.com/docker/cli), which relies upon the [moby/moby/client](https://github.com/moby/moby/tree/468eb93e5acc809248405102db32460fe7efed08/client) package for the interface to talk to _dockerd_.

## dockerd

_dockerd_ will start [listening](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/cmd/dockerd/daemon.go#L584) for user requests, and process them according to predefined [routes](https://github.com/moby/moby/tree/468eb93e5acc809248405102db32460fe7efed08/api/server/router).

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

The engine is still responsible for a variety of tasks, like interacting with [image registries](https://github.com/moby/moby/tree/468eb93e5acc809248405102db32460fe7efed08/distribution) and setting up directories on the [file system](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/daemon/daemon.go#L1204) for use by containers. The default driver will union mount an image to a directory inside of <code class="inline-highlight">/var/lib/docker/overlay2/</code>.

It is no longer responsible for managing the life cycle of running containers. As the project grew, the decision was made to split off container supervision into a separate project called _containerd_. This way, the docker daemon can continue to innovate without concern of breaking the runtime implementation.

Although [docker/engine](https://github.com/docker/engine) is forked from moby/moby, allowing for possible code divergence, they share the same commit tree to date.

### docker run

{% highlight go %}
func runContainer(dockerCli command.Cli, opts *runOptions, copts *containerOptions, containerConfig *containerConfig) error {
    ...
    // Create the container
    createResponse = createContainer(ctx, dockerCli, containerConfig, &opts.createOptions)
    ...
    // Start the container
    client.ContainerStart(ctx, createResponse.ID, types.ContainerStartOptions{});
    ...
}
{% endhighlight %}

A [<code class="inline-highlight">docker run</code>](https://github.com/docker/cli/blob/afde31d710c2057960b35332f0be6c6c2aeaf3c9/cli/command/container/run.go#L95) command begins by requesting the daemon to create a container. This request is routed to [<code class="inline-highlight">postContainersCreate</code>](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/api/server/router/container/container_routes.go#L443).

#### Create

A [couple](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/daemon/create.go#L30) [function](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/daemon/create.go#L34) [calls](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/daemon/create.go#L82) later and we're creating a container.

{% highlight go %}
func (daemon *Daemon) create(opts createOpts) (retC *container.Container, retErr error) {
    ...
    // Find the Image
    img = daemon.imageService.GetImage(params.Config.Image)
    ...
    // Create container object
    container = daemon.newContainer(opts.params.Name, os, opts.params.Config, opts.params.HostConfig, imgID, opts.managed);
    ...
    // Set RWLayer for container after mount labels have been set
    rwLayer = daemon.imageService.CreateLayer(container, setupInitLayer(daemon.idMapping))
    container.RWLayer = rwLayer
    ...
    // Create root directory 
    idtools.MkdirAndChown(container.Root, 0700, rootIDs);
    ...
    // Windows or Linux specific setup
    daemon.createContainerOSSpecificSettings(container, opts.params.Config, opts.params.HostConfig);
    ...
    // Store in a map for future lookup
    daemon.Register(container);
    ...
}
{% endhighlight %}

First we create an object to store container [metadata](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/daemon/container.go#L130).

Then like before, we create a root directory with both the image data and read-write layer inside for use by the container. Today the difference is that union mount file system support has grown to include `btrfs`, `OverlayFS` and more. To facilitate this, a [driver system](https://github.com/moby/moby/tree/468eb93e5acc809248405102db32460fe7efed08/daemon/graphdriver) abstracts away implementation.

Finally, the container object is added to the daemon's map of containers, for future use.

#### Start

The container has been created, but is not yet running. Next we [request](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/api/server/router/container/container.go#L52) to [start](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/daemon/start.go#L102) it.

{% highlight go %}
func (daemon *Daemon) containerStart(container *container.Container, checkpoint string, checkpointDir string, resetRestartManager bool) (err error) {
    ...
    // Create OCI spec for container
    spec = daemon.createSpec(container);
    ...
    // Call containerd to create the container according to spec
    daemon.containerd.Create(ctx, container.ID, spec, createOptions)
    ...
    // Call containerd to start running process inside of container
    pid = daemon.containerd.Start(context.Background(), container.ID, checkpointDir,
        container.StreamConfig.Stdin() != nil || container.Config.Tty,
        container.InitializeStdio);
    ...
}
{% endhighlight %}

This is where _containerd_ steps in, first we request a container be created according to the [OCI specification](https://github.com/moby/moby/blob/468eb93e5acc809248405102db32460fe7efed08/daemon/oci_linux.go#L695). Then, start running a process inside of the container. All subsequent supervision is handled by _containerd_.

## containerd

[_containerd_](https://containerd.io/) has confusing terminology around it. It's described as a runtime, but doesn't implement the OCI runtime spec, therefore it's not a runtime in the same way that _runc_ is. _containerd_ is a daemon which oversees the life cycle of containers, using OCI compliant runtimes in order to manage them. As [Michael Crosby](https://www.youtube.com/watch?v=VWuHWfEB6ro) describes it, _containerd_ is a container supervisor.

<img src="{{ site.baseurl }}/assets/img/docker-work/containerd.png" style="display: block; margin-left: auto; margin-right: auto;"> <center><small>(Source: Docker)</small></center>

It's designed to be the this universal base layer for supervising containers, focusing on speed and simplicity.

And it is simple, all that's required to create a container is its specification and a [bundle](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md) which encodes where the root file system is.

### Create

{% highlight go %}
func (l *local) Create(ctx context.Context, req *api.CreateContainerRequest, _ ...grpc.CallOption) (*api.CreateContainerResponse, error) {
    l.withStoreUpdate(ctx, func(ctx context.Context, store containers.Store) error {
        // Update container data store with new container record
        container := containerFromProto(&req.Container)
        created := store.Create(ctx, container)
        resp.Container = containerToProto(&created)
        return nil
    });
    ...
}
{% endhighlight %}

_dockerd_ (through a GRPC [client](https://github.com/moby/moby/blob/b9b5dc37e37e67d1cd46d9a3448c96e3f57ef4bc/libcontainerd/remote/client.go#L127)) requests _containerd_ to create a container. Upon receival, _containerd_ stores the specification in a file system backed [database](https://github.com/etcd-io/bbolt) located within <code class="inline-highlight">/var/lib/containerd/</code>.

### Start

{% highlight go %}
// Start create and start a task for the specified containerd id
func (c *client) Start(ctx context.Context, id, checkpointDir string, withStdin bool, attachStdio libcontainerdtypes.StdioCallback) (int, error) {
    ctr, err := c.getContainer(ctx, id)
    ...
    t, err = ctr.NewTask(ctx, ...)
    ...
    t.Start(ctx);
    ...
    return int(t.Pid()), nil
}
{% endhighlight %}

[Starting](https://github.com/moby/moby/blob/b9b5dc37e37e67d1cd46d9a3448c96e3f57ef4bc/libcontainerd/remote/client.go#L146) a container involves the creation and starting of a new object called a Task, which represents a process inside of a container. 

#### Task Create

{% highlight go %}
func (l *local) Create(ctx context.Context, r *api.CreateTaskRequest, _ ...grpc.CallOption) (*api.CreateTaskResponse, error) {
    container := l.getContainer(ctx, r.ContainerID)
    ...
    rtime l.getRuntime(container.Runtime.Name)
    ...
    c = rtime.Create(ctx, r.ContainerID, opts)
    ...
}
{% endhighlight %}

[Task creation](https://github.com/containerd/containerd/blob/2f60e389a03740339c9c2762004fdcb5de489b09/services/tasks/local.go#L128) is handled by the underlying container runtime. _containerd_ multiplexes OCI runtimes, therefore we need to lookup which runtime to use to create the task. The first and default runtime is _runc_. The [<code class="inline-highlight">Create</code>](https://github.com/containerd/containerd/blob/2f60e389a03740339c9c2762004fdcb5de489b09/runtime/v1/linux/runtime.go#L154) for this runtime ends up running the external process _runc_, but it does so indirectly using a _shim_.

If _containerd_ were to crash, information about running containers would be lost. To protect against this, _containerd_ [creates a management process](https://github.com/containerd/containerd/blob/2f60e389a03740339c9c2762004fdcb5de489b09/runtime/v1/shim/client/client.go#L55) for each container called a shim. The shim will call an OCI runtime to create and start a container, and then perform its duty of monitoring the container to capture the exit code and manage standard IO. 

Within [nested code](https://github.com/containerd/containerd/blob/bf5a4246798a6c1b1b0af4810fbb2d53eac91112/runtime/v1/linux/proc/init.go#L109), the shim will use [go-runc bindings](https://github.com/containerd/go-runc) to start <code class="inline-highlight">/run/containerd/runc</code> with the [create](https://github.com/containerd/go-runc/blob/e32098aae3bc878417256292705aedbde3aa3dd8/runc.go#L140) command. More on _runc_ in the next section.

In the event where _containerd_ does crash, it can recover by communicating with the shims, and reading state from <code class="inline-highlight">/var/run/containerd/</code>.

#### Task Start

Now that the container has been created, starting the task simply [directs the shim](https://github.com/containerd/containerd/blob/bf5a4246798a6c1b1b0af4810fbb2d53eac91112/runtime/v1/linux/process.go#L124) to [start the process](https://github.com/containerd/containerd/blob/bf5a4246798a6c1b1b0af4810fbb2d53eac91112/runtime/v1/linux/proc/init.go#L258) by calling [_runc_ start](https://github.com/containerd/go-runc/blob/e32098aae3bc878417256292705aedbde3aa3dd8/runc.go#L181)

## Runc

[_runc_](https://github.com/opencontainers/runc) is a command-line tool for spawning and running containers according to the OCI specification. Performing a similar job to LXC, it abstracts away the Linux kernel calls needed to create a container.

<img src="{{ site.baseurl }}/assets/img/docker-work/runc.png" style="max-width: 300px; display: block; margin-left: auto; margin-right: auto;"><center><small>(This adorable hamster belongs to runc)</small></center>

_runc_ is just one implementation of the OCI runtime spec, [many more exist](https://github.com/opencontainers/runtime-spec/blob/master/implementations.md) that can be used to create containers on a variety of systems.

### Create

{% highlight go %}
var createCommand = cli.Command{
    Name:  "create",
    Description: `The create command creates an instance of a container for a bundle. The bundle
is a directory with a specification file named "` + specConfig + `" and a root
filesystem.`,
    ...
    Action: func(context *cli.Context) error {
        spec := setupSpec(context)
        ...
        status := startContainer(context, spec, CT_ACT_CREATE, nil)
        // exit with the container's exit status so any external supervisor is
        // notified of the exit with the correct exit status.
        os.Exit(status)
        return nil
    },
{% endhighlight %}

When _runc_ [creates](https://github.com/opencontainers/runc/blob/dd50c7e3327b6c264087855f8225cb692046c5f4/create.go) a container it sets up the namespaces, cgroups and even the init process inside the container. At the end of creation, the process is paused [waiting](http://man7.org/linux/man-pages/man2/waitpid.2.html) for a signal to start running.

### Start

{% highlight go %}
var startCommand = cli.Command{
    Name:  "start",
    Usage: "executes the user defined process in a created container",
    ...
    Action: func(context *cli.Context) error {
        container := getContainer(context)
        status := container.Status()
        ...
        switch status {
        case libcontainer.Created:
            return container.Exec()
        case libcontainer.Stopped:
            return errors.New("cannot start a container that has stopped")
        case libcontainer.Running:
            return errors.New("cannot start an already running container")
        }
    },
}
{% endhighlight %}

Finally, to [start](https://github.com/opencontainers/runc/blob/dd50c7e3327b6c264087855f8225cb692046c5f4/start.go) the container, _runc_ sends a signal to the paused process to begin executing.

## The Visual Summary

In summary, launching a container using Docker 2019 involves the following steps:

{% highlight text %}
dockerd is sent POST Containers Create
    ↳ dockerd finds the requested image
    ↳ A container object is created and stored for future use
    ↳ Directories on the file system are setup for use by the container
dockerd is sent a POST Containers Start
    ↳ An OCI spec is created for the container
    ↳ containerd is contacted to create the container
        ↳ containerd stores the container spec in a database
    ↳ containerd is contacted to start the container
        ↳  containerd creates a task for the container
            ↳  The task uses a shim to call runc create
        ↳  containerd starts the task
            ↳  The task uses the shim to call runc start
        ↳ The shim / containerd continue to monitor the container until completion
{% endhighlight %}

Using _containerd's_ architecture [diagram](https://containerd.io/img/architecture.png) as a reference, we can represent the entire process visually.

<img src="{{ site.baseurl }}/assets/img/docker-work/visualsummary.png">

# Conclusion

On the surface Docker and its companion projects appear chaotic but underneath there is rigid structure and modularisation. That said, uncovering all this information was not easy, it was spread across code, blog posts, conference talks, documentation and meeting notes. Having clear "self documenting" code is a great goal to aim for, but when it comes to large systems, I don't think it's enough. Sometimes you just need to write down in plain language what a system looks like, and what each component is responsible for. 

A big thank you to all the contributors of these projects, especially those who wrote documentation which explained these systems.

I hope this has been helpful in explaining exactly how Docker runs containers. I know that I'll be coming back to read this post many more times in the future.

## Additional Sources

TODO

https://www.youtube.com/watch?v=sK5i-N34im8s
https://blog.docker.com/2016/04/docker-containerd-integration/
https://www.youtube.com/watch?v=ZAhzoz2zJj8
https://blog.docker.com/2017/08/what-is-containerd-runtime/
