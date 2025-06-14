## Introduction

This repository holds Docker/Podman container base image build scripts
that let you to build a stable general purpose development environment
that can be quickly extended as needed for specific use cases.

Both [Docker][Docker] and [Podman][Podman] container environments are
supported under Windows, Linux, and MacOS with the same scripts so it
really doesn't matter which environment you choose.

Podman can read Docker files, and tools like VSCode already know
how to use Docker files, so we maintain the convention of using `docker` in
those filenames, even when we are building images and containers using Podman.

I won't recommend one over the other, it's up to you to choose what
makes the most sense for your personal or business environment.

The following terms are useful to keep in mind:

| Term         | Description |
| -            | -           |
| Image        | The template for a container - you can re-use the same image for multiple containers
| Container    | An instance of an image set up for a specific task
| Devcontainer | A special type of container that VSCode can interact with from the host machine
| Volume       | A file system that can be attached to a running container

:exclamation: If you are using Podman, then you will need to make a
few adjustments to be able to use the Docker support plugin and to
connect from the container to a network port on the host machine. This
is described in [Podman Specific Setup for Windows](#Podman-Specific-Setup-for-Windows)
near the end of this document.

## Install a Container Environment

If you have not already done so, choose and install a containerization
environment.

- [Docker Windows Install]
- [Podman Windows Install]

Read and follow the instructions for using the WSL system with your
containerization environnment, including any computer restarts.

### Post--Install Insrtuctions for Podman

For the highest level of security, we want to run our Podman containers
as rootless and with no user mode networking. Open a command prompt and
run:

```
podman machine stop
podman machine set --rootful=false
podman machine set --user-mode-networking=true
podman machine start
```

## Docker/Podman Base Image Builds

We are now ready to begin building a base image that can be used to
build a devcontainer. Why do it like this? Mainly to speed up the
creation of multiple devcontainers using a common base image.

For example, I do embedded systems development in C, and also
tasks like writing for my website or 3D modelling that use
Python but don't need the C compiler. It makes sense to have
two base images, one for embedded development and one for Python.
I can then quickly create a task-specific devcontainer with just a few
additions.

Containers are awesome tools to simplify your development process. To
learn more about how they work, I can recommend [this 'zine by Julia
Evans][how-containers-work].

If you have ever wondered how containers get their default names, then
[this file][container-names] will be interesting.

### Starting a Base Image Build

Docker and Podman have almost identical command lines. This is by design as
Podman came after Docker, and rather than design yet another almost compatible
command line, the Podman team wisely chose to simply duplicate the semantics
of Docker. They even went so far as to be able to read the Dockerfile format.

| Environment | Version | Notes |
| -            | - | - |
| Docker       | 4.32.0+ | The traditional `build` command is now legacy, so we will be using `buildx` instead. If you need more control over the process, refer to the [Docker CLI reference for `buildx`][Docker buildx].
| Podman       | 1.16.2+ | Podman also supports `buildx` - but not all of the features from Docker are available. [Podman CLI reference for `build`][Podman build].

:exclamation: In each of the code sections below, you will see two command
lines, one for the Docker environment, and one for Podman - choose
the one suitable for your environment.

:warning: Resist the urge to customize these base images, we are going to
make use-case specific changes when we build the devcontainer from the
base image.

:warning: In the examples, we refer to the Ubuntu version as `xx.yy`.
Use the actual version when you type the command, for example 22.04.

:warning: On Windows, you will need to use the `\` as a path separator.

The simplest way to kick off the build of a new image is:

```
cd path/to/ubuntu-xx.yy-embedded

# Choose podman or docker depending on your containerization tool
#
docker buildx build -f Dockerfile -t ubuntu-xx.yy-embedded .
podman buildx build -f Dockerfile -t ubuntu-xx.yy-embedded .

NOTE: Don't forget the trailing "."
```

If you want to avoid changing to the directory where the `Dockerfile`
lives, you can just do this:

```
# Choose podman or docker depending on your containerization tool
#
docker buildx build -t ubuntu-xx.yy-embedded path/to/ubuntu-xx.yy-embedded
podman buildx build -t ubuntu-xx.yy-embedded path/to/ubuntu-xx.yy-embedded
```

Depending on network and computer speed, the build time will vary. On my
laptop it's about 5 minutes without the cache - and not even 5 seconds with
the cache.

### Renaming Base Images

Building a base image from the instructions above will result in
an image with the name `ubuntu-xx.yy-embedded:latest`. The tag
defaults to `latest`. But what if you want
to change that name?

The full name of an image is also called a tag, and you can create as
many tags as you want for an image without using any more disk space.
If you need more control over the tag process, refer to the
[Docker CLI reference for `tag`][Docker tag] or
[Podman CLI reference for `tag`][Podman tag].

You rename/tag an image like this:

```
# Choose podman or docker depending on your containerization tool
#
docker image tag ubuntu-22.04-embedded some-new-name:some-new-tag
podman image tag ubuntu-22.04-embedded some-new-name:some-new-tag
```

Sometimes the orignal tag is a really long string, for example a VSCode
devcontainer has a hash after the name.

`vsc-container-name-614c6be1beb9a2a0db595c65c0d81774b6cd195b6092ff5ca691bd219d5caaa6:latest`

That's not going to be fun, even with copy/paste. So we can substitute
the Image ID for the name, which can be copied from the Docker Desktop
Image tab, or from the command line:

```
docker image list
podman image list

REPOSITORY                      TAG    IMAGE ID     CREATED        SIZE
ubuntu-22.04                    22.04  3b63e27dcd9f 31 minutes ago 2.24GB
vsc-container-name-614c6be1b... latest 047565335b81 16 hours ago   2.25GB
plantuml/plantuml-server        jetty  cba36f940161 6 weeks ago    483MB
...
```

So making that VSCode Image more friendly looks like this:

```
# Choose podman or docker depending on your containerization tool
#
docker image tag 047565335b81 some-new-name:some-new-tag
podman image tag 047565335b81 some-new-name:some-new-tag
```

## `apt` Pinning - Ensure Secure and Repeatable Images

One of the problems we are trying to solve with containers is a repeatable
build environment. This can avoid "it works on my machine" problems and
keeps the development team happy. It also keeps the legacy support team
happy because they can recreate the build environment.

This is true for as long as DockerHub is around, and for as long as
the the Linux distro provides a package manager, and for as long
as package maintainers keep the packages available.

For this reason, it is HIGHLY recommended that your team keeps an
archive of mission critical Docker/Podman images and containers somewhere
safe. For more details on saving images read the
[Docker CLI reference for `image save`][Docker image save]
or the
[Podman CLI reference for `image save`][Podman image save]

The `Dockerfile` is where we specify which packages are to be added
the base Ubuntu image - but how do we ensure that the package versions
stay stable, and still receive security updates? The secret lies
in [`apt` pinning][apt-pinning]

We want to make sure that specific versions of the essential packages
are retreived, but the Ubuntu LTS releases do provide security updates
from time to time. In the past this meant updating the `Dockerfile` with
more specific version strings, but now we encapsulate this in the
`apt.preferences` file that is copied to `/etc/apt` in the image creation
step.

A good example is the `ssh` package, it's at the heart of your container
network access security management. Here is the section from
`apt.preferences` for that package:

```
Package: ssh
Pin: version 1:8.9*
Pin-Priority: 1000
```

This says that version `1:8.9` followed by any additional characters
will be installed. The whole point of the `apt` package manager is
to keep your system up to date. Currently the preferred version of the
`ssh` package for Ubuntu 22.04 (jammy) is:

```
jammy (22.04LTS) (net): secure shell client and server (metapackage)
1:8.9p1-3ubuntu0.10 [security]: all
```

When a security update is made, any old package versions are *deleted* from
the Ubuntu package manager - this makes sense because you don't want to
install a package version with a known security issue.

As long as the Ubuntu Jammy package maintainers keep the version of
`ssh` at `8.9` we are good. Ubuntu 22.04 LTS is by definition the Long Term
Support version, so we can be confident that the version number of `ssh`
is unlikely to change, except for security patches.

## Podman Specific Setup for Windows

There are two relatively simple changes required to be able to use
Podman instead of Docker for your container management.

### VSCode Support for Podman in the Remote Explorer Plugin

The [VSCode Remote Explorer] plugin works together with the
[VSCode DevContainer] plugin to give you visibility into your
container infrastructure - you do not need the [VSCode Docker] plugin
to get this functionality.

The [VSCode DevContainer] plugin assumes that your default container management
tool is Docker. Thanks to the Podman team deciding to use the same command-line
interface as Docker, we can simply set the name of the container management
executable.

Go to Settings -> Dev -> Containers -> DockerPath and set the
field to "podman"

Or, just add this to your `settings.json`:

```
"dev.containers.dockerPath": "podman",
```

Restart VSCode and now your Remote Explorer plugin will call `podman`
to show you all of your container assets, and the DevContiner plugin
will use `podman` to start, stop, and rebuild containers.

### Podman Container -> Windows File System Metadata

Mounting a Windows filesystem in a container is a great way to share files
between the host and the container, but you will not be able to change the
file permissions on the Windows host files from the container. This makes
sharing things like your user's `.ssh` directory problematic because it
will have the wrong permisisons by default.

To allow the Windows filesystem permisison metadata to be changed by the
host we need to edit the `/etc/wsl.conf` file on the machine that the
container management system is running on.

You can find your default WSL container machine from the Windows command
promopt with:

```
C:\Users\YourUserID> wsl --list

Windows Subsystem for Linux Distributions:
podman-machine-default (Default)
podman-net-usermode
```

Start the WSL machine, if it's not already running, open a terminal in the
machine and add the following lines to `/etc/wsl.conf`:

```
[automount]
options = metadata
```

If there is already an `[automount]` section then just add the `options = metadata`
line, and if there is already an `options` key, then make sure `metadata` is in
the comma separated list of options.

You must restart your WSL machine for the changes to take effect.

You can always refer to the [WSL Configuration Reference Page][WSL Configuration] for
more details.


### Podman Container -> Host Networking in Windows

The Podman network name resolution works differently than Docker, so
we will need to do some one-time configuration to get around it.

Each Podman machine (there is typically only one) gets a unique IP
address, and you can retreive it from a PowerShell session (your IP
address will be different):

```
Get-NetIpAddress | where { $_.InterfaceAlias -Like '*WSL*' -and $_.AddressFamily -EQ "IPv4" } |select -ExpandProperty IPAddress
172.17.32.1
```

Now start your Podman machine:

```
wsl --distribution podman-machine-default
```

and edit (using `sudo vi`) the `/etc/containers/containers.conf` file
to add this line in the `[containers]` section. This will allow
`host.containers.internal` to resolve to the IP address that the
host presents to `podman-machine-default` - of course your IP
address will be different.

```
host_containers_internal_ip = "172.17.32.1"
```

It turns out that you can use additional hostnames if you already have
devcontainer files that use a different hostname. For example, in my
devcontainers for embedded developemnt, I run [OpenOCD] and connect
to it from the container using the name `host.gdb.gateway`. So the
`[containers]` section of my `/etc/containers/containers.conf` looks
like this:

```
[containers]
host_containers_internal_ip = "172.17.32.1"
host_docker_internal_ip = "172.17.32.1"
host_gdb_gateway_ip = "172.17.32.1"
```

To translate your IP name to the corresponding variable name in the `[containers]` section, just replace the "." with "_" and add "_ip".

Remember to restart your Podman machine after making any changes:

```
podman machine stop
podman machine start
```

Thanks to [StackOverflow user Anton][StackOverflow Anton] for
[this StackOverflow solution for Podman host networking support][podman host networking].

## Verifying Host <-> Container Networking

The simplest way to verify if the Host <-> Container networking is functional
is to set up a web server on the host, and try to `wget` from the container.

On your host machine type the following in a terminal:

```
python -m http.server
```

This will start a server that binds to ALL interfaces at port 8000.

Now start your container, and open up a terminal in the container. There is
usually a menu in your container GUI that lets you open a terminal in a
running container.

To see the hostnames your container supports:

```
yourUser@083312b7009d:/$ cat /etc/hosts
172.17.32.1     host.gdb.gateway
...
172.17.32.1     host.containers.internal host.docker.internal
yourUser@083312b7009d:/$
```

Recall that you set up your host IP address in the default machine in
a previous step, and here it is again. You should now be able to
query the webserver we left running on the host in ANY of these ways:

```
wget 172.17.32.1:8000
wget host.gdb.gateway:8000
wget host.containers.internal:8000
wget host.docker.internal:8000
```

The response looks like this if Host <-> COntainer networking is
functional:

```
yourUser@083312b7009d:~/projects$ wget 172.17.32.1:8000
--2025-05-27 22:39:41--  http://172.17.32.1:8000/
Connecting to 172.17.32.1:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 654 [text/html]
Saving to: ‘index.html’

index.html                         100%[================================================================>]     654  --.-KB/s    in 0s      

2025-05-27 22:39:41 (72.2 MB/s) - ‘index.html’ saved [654/654]

yourUser@083312b7009d:~/projects$ 
```

## Troubleshooting Build Issues

Sometimes the image won't build. Here are a few common failures:

### Failure to Download Packages

We are building on the official Ubuntu images from DockerHub, and in
turn those images will update packages from other servers. Sometimes
those servers are down, in which case trying later helps.

If you are using an externally managed computer, then VPN or other
network settings may be preventing access to certain IP addresses. Consult
your provider for instructions on how to proceed to access external
IP addresses.

For example, you might run into trouble downloading the `plantuml` file
from `github.com`:

```
0.570 Resolving objects.githubusercontent.com ...
0.630 Connecting to objects.githubusercontent.com ...
0.795 ERROR: cannot verify objects.githubusercontent.com's certificate ...
0.795   Unable to locally verify the issuer's authority.
0.795 To connect to objects.githubusercontent.com insecurely, use `--no-check-certificate'.                             -
```

Yeah, don't start modifying the `Dockerfile` - the problem is most likely
your network settings. In my case I just had to (temporarliy) disable the
`ZScaler` Internet Security setting on my managed laptop. Note that it
was still on the VPN and that our organization sets a timeout so that
if you forget to turn it back on you are still safe.

### Cache Confusion

Having a cache of previously created container layers really helps to
speed up container construction - but sometimes the cache works against
you. One way around this is to force a build without the cache:

```
# Choose podman or docker depending on your containerization tool
#
docker buildx build --no-cache -t ubuntu-xx.yy-embedded path/to/ubuntu-xx.yy-embedded-docker
podman buildx build --no-cache -t ubuntu-xx.yy-embedded path/to/ubuntu-xx.yy-embedded-docker
```

As an absolute last resort, you can consider clearing out your entire
Docker cache - this can actually free up quite a bit of disk space, but you
will slow down any subsequent container builds until the cache is refilled.

```
docker buildx prune -f
```

Podman users will need to manage unused images, containers, and volumes either
with the GUI or command lines.

## More Detailed Information

Managing users in rootless mode
https://medium.com/@guillem.riera/making-visual-studio-code-devcontainer-work-properly-on-rootless-podman-8d9ddc368b30

[Docker]: https://docs.docker.com
[Podman]: https://podman.io/
[Docker buildx]: https://docs.docker.com/reference/cli/docker/buildx/build
[Podman build]: https://docs.podman.io/en/latest/markdown/podman-build.1.html
[Docker Windows Install]: https://docs.docker.com/desktop/setup/install/windows-install/
[Podman Windows Install]: https://podman-desktop.io/docs/installation/windows-install
[Docker tag]: https://docs.docker.com/reference/cli/docker/image/tag
[Podman tag]: https://docs.podman.io/en/latest/markdown/podman-tag.1.html
[Docker image save]: https://docs.docker.com/reference/cli/docker/image/save
[Podman image save]: https://docs.podman.io/en/latest/markdown/podman-save.1.html
[apt-pinning]:  https://help.ubuntu.com/community/PinningHowto
[how-containers-work]: https://jvns.ca/blog/2020/04/27/new-zine-how-containers-work/
[container-names]: https://github.com/moby/moby/blob/master/pkg/namesgenerator/names-generator.go
[VSCode Remote Explorer]: https://marketplace.visualstudio.com/items?itemName=ms-vscode.remote-explorer
[VSCode DevContainer]: https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers
[VSCode Docker]: https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker
[podman host networking]: https://stackoverflow.com/questions/79098571/podman-container-cannot-connect-to-windows-host
[WSL Configuration]: https://learn.microsoft.com/en-us/windows/wsl/wsl-config
[StackOverflow Anton]: https://stackoverflow.com/users/5788429/anton
[OpenOCD]: https://openocd.org/
[What is Podman]: https://www.redhat.com/en/topics/containers/what-is-podman#why-podman