<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

  - [What are containers?](#what-are-containers)
  - [Why do we need containers?](#why-do-we-need-containers)
- [Docker Overview](#docker-overview)
- [Docker Architecture](#docker-architecture)
- [Working with Docker](#working-with-docker)
  - [Running a container](#running-a-container)
  - [Building our own image](#building-our-own-image)
    - [ENTRYPOINT vs CMD](#entrypoint-vs-cmd)
  - [Understanding Networks](#understanding-networks)
  - [Persisting data](#persisting-data)
    - [Bind mount](#bind-mount)
    - [Volume mount](#volume-mount)
- [Some useful reads](#some-useful-reads)
  - [Docker container escapes](#docker-container-escapes)
  - [Tutorials](#tutorials)
  - [Linux](#linux)
  - [Containers and image initiatives](#containers-and-image-initiatives)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## What are containers?

Containers are `isolated` group of processes, running on a `single host`, that does a specific task.

`Containerization` was possible because of linux concepts like [namespcaes][man_namespace] and [cgroups][man_cgroups].

- namespaces : controls what you see.
- cgroups : controls what you use.

Containers allow us to package applications along with libraries and other requirements, so that they can be run on any machine reliably.

VMs allow us to run more than one OS on a host OS, whereas containers are run on a single host OS and share the same OS kernel and resources.

![alt vm vs container](vm_containers.png)

If you are interested in knowing more about containers and how isolation is acheived, check out this series.

[Demistifying containers: Part 1][undersanding_containers_1]

[Demistifying containers: Part 2][undersanding_containers_2]

[Demistifying containers: Part 3][undersanding_containers_3]

Also this is an amazing video on [building a container from scratch](https://www.youtube.com/watch?v=8fi7uSYlOdc&ab_channel=GOTOConferences)

## Why do we need containers?

Compared to VMs, containers are lightweight and consume less resources.

They are isolated, can be run anywhere and also provide a consistent environment for runnin them.

This allows developers to easily test and deploy the application without having to worry about the sandbox and production environments and version of the dependencies.

# [Docker Overview](https://docs.docker.com/get-started/overview/)

Docker is a container engine technology written in Golang, which makes use of [LXC (LinuX Containers)](https://wiki.archlinux.org/title/Linux_Containers).

Can only be run on linux platform. Mac and Windows run docker inside a linux VM

Checout the [docker container manifesto][container_manifesto].

> Note: Docker is not the only container engine out there like LXC, LXD, runc and crictl. There are several others. What is really important is the concept of containers and images, docker is just a tool that enables us to use them.

# Docker Architecture

![alt architecture](docker_architecture.svg)

# Working with Docker

Lets understand docker with examples.

## Running a container

To run a container, we need an image. An image is nothing but a rootfs, with all your dependencies, libraries and environment variables required to run your application. For this example, lets use an `ubuntu` image. We can run this image in a container using the `docker run` command.

```bash
docker pull ubuntu // skippable

docker run ubuntu
```

We can skip the `docker pull` step and directly run `docker run`.
This should print something like,

```
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
c549ccf8d472: Already exists
Digest: sha256:aba80b77e27148d99c034a987e7da3a287ed455390352663418c0f2ed40417fe
Status: Downloaded newer image for ubuntu:latest
```

Notice how it checks local and then tries to fetch from remote if it doesn't exits.

Now lets check if the container is running.

```bash
docker ps
```

Wait, there is nothing there.

> Note: By default, docker ps lists only running containers, by passing a `-a` flag we are making it display all containers.

```bash
docker ps -a
```

Seems like the container exited just after we ran it.

Lets try running a process in the image and check `docker ps`

```bash
docker run -d node:alpine sleep 10
docker ps
```

We have a container that is running now. But why did it exit the first time

> Containers are supposed to run groups of tasks or processes/tasks.
> Once the task/ process is complete, the container exits

Running an image with `docker run -d` runs it in detached mode. You can re-attach the container to a terminal using `docker attach CONTAINER_ID` command.

## Building our own image

If you want to checkout what is inside the ubuntu image, try

```bash
docker run -t ubuntu
```

This runs the ubuntu image and attaches a terminal to it. This should also be visible under `docker ps` also.

Lets look at some useful commands that we can use on images.

```bash
docker history IMAGE_TAG
```

This gives you a nice history of all the modifications you made to the image.

```bash
docker inspect IMAGE_TAG
```

This gives information about the image(env variables, cmd, rootfs...)

Now lets build our own image. For this example, lets try running a js program which prints `hello world`.

```Dockerfile
FROM node:alpine
COPY . /app
CMD [ "node","index.js" ]
```

This is saved as `Dockerfile` in the directory where the source code is. And will be used to build an image.

Lets deconstruct this Dockerfile, and see what each statement does.

`FROM node:alpine` : using a node:alpine image as the base to build on top of

`COPY . /app` : copying all the files to a `/app` directory

`CMD [ "node","index.js" ]` : executing a node command to run `index.js`.

### ENTRYPOINT vs CMD

An `ubuntu` image which sleeps for 5 seconds.

```
docker run -i ubuntu-sleeper sleep 10
```

Same using a docker file.

```Dockerfile
FROM ubuntu
CMD [ "sleep", "5" ]
```

Build and run the image

```bash
docker build -t ubuntu-sleeper .
docker run -i ubuntu-sleeper
```

What if we want to pass the time as an argument. This can be done using the `ENTRYPOINT`command. This defines the base command which will be run in the image.

> There can only be one ENTRYPOINT/ CMD in a Dockerfile.

```Dockerfile
FROM ubuntu
ENTRYPOINT [ "sleep" ]
# CMD [ "5" ]
```

```bash
docker build -t ubuntu-sleeper .
docker run -i ubuntu-sleeper 10
```

You can also do without a Dockerfile like this,

```
docker run --entrypoint sleep ubuntu 10
```

https://design.jboss.org/redhatdeveloper/marketing/docker_cheatsheet/cheatsheet/images/docker_cheatsheet_r3v2.pdf

## Understanding Networks

Docker provides three types of networks.

- Bridge : Docker maintains its own network with an internal DNS.
- None : Network is isolated and will not be visible to the host.
- Host : Network is mapped to the host network.
- overlay : Communication over several docker hosts.
- macvlan : Assigns a MAC address to the containers and they behave like physical devices.

```bash
docker network list
```

```bash
docker network create \
    --driver bridge \
    --subnet 172.18.0.0/16
    sleeper-network
```

> The default network is bridge

```
docker build -t flask-hello ./flask/app/
docker run -i -p 5000:5000 flask-hello
```

```
docker build -t flask-hello ./flask/app/
docker run -i --network=host -p 5000:5000 flask-hello
```

## Persisting data

Whatever data we have inside the container, will be lost once the container exists.
There are two ways to persist data.

- Bind mount and
- Volume mount

![alt mounting](types-of-mounts-volume.png)

### Bind mount

Bind mount mounts a file on the host system to the container.

```
docker run -i --volume ./data:/var/lib/postgresql/data postgres
```

### Volume mount

Volumes are managed by docker.

```
docker volume create pg_data
docker run -i --volume pg_data:/var/lib/postgresql/data postgres
# docker run -i --mount  source=pg_data,target=/var/lib/postgresql/data postgres
```

# Some useful reads

## Docker container escapes

https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/
https://betterprogramming.pub/escaping-docker-privileged-containers-a7ae7d17f5a1

## Tutorials

https://www.youtube.com/watch?v=pTFZFxd4hOI&ab_channel=ProgrammingwithMosh
https://www.youtube.com/watch?v=fqMOX6JJhGo&ab_channel=freeCodeCamp.org

## Linux

https://man7.org/linux/man-pages/man2/chroot.2.html
https://man7.org/linux/man-pages/man2/pivot_root.2.html
https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt

## Containers and image initiatives

https://www.cncf.io/
https://opencontainers.org/
https://github.com/opencontainers/runtime-spec
https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/
https://www.tutorialworks.com/difference-docker-containerd-runc-crio-oci/

[man_namespace]: https://man7.org/linux/man-pages/man7/namespaces.7.html
[man_cgroups]: https://man7.org/linux/man-pages/man7/cgroups.7.html
[manifesto]: https://github.com/moby/moby/blob/0db56e6c519b19ec16c6fbd12e3cee7dfa6018c5/README.md
[undersanding_containers_1]: https://medium.com/@saschagrunert/demystifying-containers-part-i-kernel-space-2c53d6979504
[undersanding_containers_2]: https://medium.com/@saschagrunert/demystifying-containers-part-ii-container-runtimes-e363aa378f25
[undersanding_containers_3]: https://medium.com/@saschagrunert/demystifying-containers-part-iii-container-images-244865de6fef
