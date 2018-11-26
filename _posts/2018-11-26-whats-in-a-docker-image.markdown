---
layout: post
title:  "What's in a Docker image?"
date:   2018-11-26
---

It's a good question, and before you know the answer, Docker images can seem pretty mysterious. Not only do I want to give you the answer, but I want to show you how I got there.

# From a Dockerfile to an Image

Let's start at the beginning. Hopefully you're all familiar with a Dockerfile - the instructions on how Docker will build an image for you. Here's a simple example.

{% highlight text %}
FROM ubuntu:15.04
COPY app.py /app
CMD python /app/app.py
{% endhighlight %}

Each of these lines are instructions to Docker on how to build an image. It will use <code class="inline-highlight">ubuntu:15.04</code> as the base and then copy in a python script. The <code class="inline-highlight">CMD</code> instruction is a directive on what to do when you run the container (turn the image into a running process), and therefore not relevant at this stage.

Let's run <code class="inline-highlight">docker build .</code> and check the output.

{% highlight text %}
$ docker build -t my_test_image .
Sending build context to Docker daemon  364.2MB
Step 1/3 : FROM ubuntu:15.04
 ---> d1b55fd07600
Step 2/3 : COPY app.py /app/
 ---> 44ab3f1d4cd6
Step 3/3 : CMD python /app/app.py
 ---> Running in c037c981012e
Removing intermediate container c037c981012e
 ---> 174b1e992617
Successfully built 174b1e992617
Successfully tagged my_test_image:latest
{% endhighlight %}

Looking at the last two lines, we have succesfully built a Docker image which we can refer to by the identifier <code class="inline-highlight">174b1e992617</code> (This value is a SHA256 hash digest of the image contents).

We have our final image, but what are the IDs from the individual steps? <code class="inline-highlight">d1b55fd07600</code> and <code class="inline-highlight">44ab3f1d4cd6</code>? Are they images aswell? Actually, yes, they are. Imagine if we got rid of Step 2 (<code class="inline-highlight">COPY app.py /app</code>) from our Dockerfile, Docker would still succesfully build that as an image (ignoring the fact that the CMD would fail because <code class="inline-highlight">app.py</code> is missing). So at each step in the image building process, we have an image. 

Which tells us that images can be built ontop of each other! That makes sense when you consider the <code class="inline-highlight">FROM</code> directive in the Dockerfile is just specifying which image to build ontop of.

The structure of an image must be organised in such a way to allow this, but how? We're going to pull apart a docker image to find out.

# Exporting an image, and unpacking it

For ease of use, images can be exported to a single file, making it simple for us to take a look inside.

<code class="inline-highlight">docker save my_test_image > my_test_image</code>

And the exported file is....

{% highlight text %}
$ file my_test_image
my_test_image: POSIX tar archive
{% endhighlight %}

A tarball! A compressed file or directory. Let's unpack it.

{% highlight text %}
$ mkdir unpacked_image
$ tar -xvf my_test_image -C unpacked_image
x 174b1e9926177b5dfd22981ddfab78629a9ce2f05412ccb1a4fa72f0db21197b.json
x 28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/
x 28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/VERSION
x 28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/json
x 28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/layer.tar
x 4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/
x 4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/VERSION
x 4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/json
x 4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/layer.tar
x 6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/
x 6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/VERSION
x 6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/json
x 6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/layer.tar
x c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/
x c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/VERSION
x c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/json
x c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/layer.tar
x cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/
x cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/VERSION
x cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/json
x cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/layer.tar
x manifest.json
x repositories
{% endhighlight %}

We will start our investigation at <code class="inline-highlight">manifest.json</code>

{% highlight json %}
[
  {
    "Config": "174b1e9926177b5dfd22981ddfab78629a9ce2f05412ccb1a4fa72f0db21197b.json",
    "RepoTags": [
      "my_test_image:latest"
    ],
    "Layers": [
      "cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/layer.tar",
      "28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/layer.tar",
      "4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/layer.tar",
      "c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/layer.tar",
      "6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/layer.tar"
    ]
  }
]
{% endhighlight %}

The manifest file is a piece of metadata which describes exactly what's inside this image. We can see the the image has a tag <code class="inline-highlight">my_test_image</code>, and it has something called Layers and another called Config.

The first 12 characters of the config JSON file is the same as the image id we saw from the docker build, coincidence - I think not!

{% highlight json %}
$ cat 174b1e9926177b5dfd22981ddfab78629a9ce2f05412ccb1a4fa72f0db21197b.json
{
  "architecture": "amd64",
  "config": {
    "Hostname": "d2d404286fc4",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh",
      "-c",
      "python /app/app.py"
    ],
    "ArgsEscaped": true,
    "Image": "sha256:44ab3f1d4cd69d84c9c67187b378b1d1322b5fddf4068c11e8b11856ced7efc0",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": null,
    "OnBuild": null,
    "Labels": null
  },
  "container": "c037c981012e8f03ac5466fcdda8f78a14fb9bb5ee517028c66915624a5616fa",
  "container_config": {
    "Hostname": "d2d404286fc4",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh",
      "-c",
      "#(nop) ",
      "CMD [\"/bin/sh\" \"-c\" \"python /app/app.py\"]"
    ],
    "ArgsEscaped": true,
    "Image": "sha256:44ab3f1d4cd69d84c9c67187b378b1d1322b5fddf4068c11e8b11856ced7efc0",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": null,
    "OnBuild": null,
    "Labels": {}
  },
  "created": "2018-11-01T03:19:16.8517953Z",
  "docker_version": "18.09.0-ce-beta1",
  "history": [
    {
      "created": "2016-01-26T17:48:17.324409116Z",
      "created_by": "/bin/sh -c #(nop) ADD file:3f4708cf445dc1b537b8e9f400cb02bef84660811ecdb7c98930f68fee876ec4 in /"
    },
    {
      "created": "2016-01-26T17:48:31.377192721Z",
      "created_by": "/bin/sh -c echo '#!/bin/sh' > /usr/sbin/policy-rc.d \t&& echo 'exit 101' >> /usr/sbin/policy-rc.d \t&& chmod +x /usr/sbin/policy-rc.d \t\t&& dpkg-divert --local --rename --add /sbin/initctl \t&& cp -a /usr/sbin/policy-rc.d /sbin/initctl \t&& sed -i 's/^exit.*/exit 0/' /sbin/initctl \t\t&& echo 'force-unsafe-io' > /etc/dpkg/dpkg.cfg.d/docker-apt-speedup \t\t&& echo 'DPkg::Post-Invoke { \"rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true\"; };' > /etc/apt/apt.conf.d/docker-clean \t&& echo 'APT::Update::Post-Invoke { \"rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true\"; };' >> /etc/apt/apt.conf.d/docker-clean \t&& echo 'Dir::Cache::pkgcache \"\"; Dir::Cache::srcpkgcache \"\";' >> /etc/apt/apt.conf.d/docker-clean \t\t&& echo 'Acquire::Languages \"none\";' > /etc/apt/apt.conf.d/docker-no-languages \t\t&& echo 'Acquire::GzipIndexes \"true\"; Acquire::CompressionTypes::Order:: \"gz\";' > /etc/apt/apt.conf.d/docker-gzip-indexes"
    },
    {
      "created": "2016-01-26T17:48:33.59869621Z",
      "created_by": "/bin/sh -c sed -i 's/^#\\s*\\(deb.*universe\\)$/\\1/g' /etc/apt/sources.list"
    },
    {
      "created": "2016-01-26T17:48:34.465253028Z",
      "created_by": "/bin/sh -c #(nop) CMD [\"/bin/bash\"]"
    },
    {
      "created": "2018-11-01T03:19:16.4562755Z",
      "created_by": "/bin/sh -c #(nop) COPY file:8069dbb6bfc301562a8581e7bbe2b7675c2f96108903c0889d258cd1e11a12f6 in /app/ "
    },
    {
      "created": "2018-11-01T03:19:16.8517953Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/sh\" \"-c\" \"python /app/app.py\"]",
      "empty_layer": true
    }
  ],
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:3cbe18655eb617bf6a146dbd75a63f33c191bf8c7761bd6a8d68d53549af334b",
      "sha256:84cc3d400b0d610447fbdea63436bad60fb8361493a32db380bd5c5a79f92ef4",
      "sha256:ed58a6b8d8d6a4e2ecb4da7d1bf17ae8006dac65917c6a050109ef0a5d7199e6",
      "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
      "sha256:9720cebfd814895bf5dc4c1c55d54146719e2aaa06a458fece786bf590cea9d4"
    ]
  }
}
{% endhighlight %}

It's quite a big JSON file but looking through you can see that there's lots of different metadata in here. In particular, there is metadata about how to turn this image into a running container - the command to run and environment variables to add.

# Images are like Onions

They both have layers. But what's a layer? I'm going to pick on 
<code class="inline-highlight">cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb</code> because that was the first one in the Layers list.

{% highlight text %}
$ ls cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb
VERSION   json      layer.tar
{% endhighlight %}

Another tarfile, let's unpack it and take a look.

{% highlight text %}
$ tree -L 1
.
├── bin
├── boot
├── dev
├── etc
├── home
├── lib
├── lib64
├── media
├── mnt
├── opt
├── proc
├── root
├── run
├── sbin
├── srv
├── sys
├── tmp
├── usr
└── var
{% endhighlight %}

And this is the big secret of Docker images, it's made up of different views of the file system! There's quite a lot in this layer, userland binaries in <code class="inline-highlight">/bin</code>, shared libraries in <code class="inline-highlight">/usr/lib</code>, almost everything you would see looking around in a standard Ubuntu filesystem. 
So what does each layer contain exactly? Well it would help to know which layers came from the base image, and which were added by us. 

Using the same process we did earlier but on <code class="inline-highlight">ubuntu:15.04</code> I can see that layers 

{% highlight text %}
cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb
28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114
4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e
c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c
{% endhighlight %}

all belong to the ubuntu base image, the <code class="inline-highlight">FROM ubuntu:15.04</code> command. Knowing this, I predict that the top most layer of our <code class="inline-highlight">my_test_image</code> image, <code class="inline-highlight">6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373</code>, should be from the command <code class="inline-highlight">COPY app.py /app/</code>.

{% highlight text %}
$ tree
.
└── app
    └── app.py
{% endhighlight %}

It is, and all that's inside is the change that we made to the filesystem, which was just adding the <code class="inline-highlight">app.py</code> file.

## Visual Aid

To see it all come together visually, our image looks like this:

<img src="{{ site.baseurl }}/assets/img/docker-image/layers.png">

## Bonus Round

Doing that manually was quite a bit of effort, but it's rewarding to do it at least once. If you ever want to analyse your images in the future, you can use the Open Source tool [dive](https://github.com/wagoodman/dive)!

<img src="{{ site.baseurl }}/assets/img/docker-image/dive.png">

# How does this get turned into a running container?

Now that we understand what a Docker image is, how does Docker turn this into a running container?

## File System

Each container has it's own view of the filesystem, Docker will take all of the layers inside the image and lay them ontop of each other to present one view of the filesystem. This technique is called [Union Mounting](https://en.wikipedia.org/wiki/Union_mount), Docker supports several Union Mount Filesystems on Linux, the main ones being [OverlayFS](https://en.wikipedia.org/wiki/OverlayFS) and [AUFS](https://en.wikipedia.org/wiki/Aufs).

But that's not all, Containers are meant to be ephemeral, changes to the filesystem while the container is running should not be saved once the container stops. One way to do this could be to copy the entire image somewhere else, that way the changes will not affect the original files. This is not very efficient, the alternative (and what Docker does) is to add a thin Read/Write layer to the very top of the filesystem in the container where the changes will be made. If you need to make a change to a file in a layer below, that file will need to be copied up to the top layer where changes are made. This is called [Copy-On-Write](https://en.wikipedia.org/wiki/Copy-on-write). When the container stops running, the top most file system layer is discarded.

This image from the [docker documentation](https://docs.docker.com/v17.09/engine/userguide/storagedriver/imagesandcontainers/#images-and-layers) shows the topmost layer.

![](https://docs.docker.com/v17.09/engine/userguide/storagedriver/images/container-layers.jpg)

## Everything else

The full process of starting a container is out of scope for this article. After the filesystem, the image is not used for much else other than its metadata configuring some of the next steps. For completeness, to make a running container we need to use [_Namespaces_](https://en.wikipedia.org/wiki/Linux_namespaces) to control what the process can see (Filesystem, Processes, Network, Users, etc); Use [_cgroups_](https://en.wikipedia.org/wiki/Cgroups) to control what resources a process can use (Memory, CPU, Network, etc); and _Security Features_ to control what a process can do (Capabilities, AppArmor, SELinux, Seccomp).
