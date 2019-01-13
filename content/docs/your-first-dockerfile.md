---
title: "Your First Dockerfile"
date: 2019-01-10T12:04:52+03:00
draft: false
---

# Your First Dockerfile

**Jan 13, 2019**\
<!-- <sup>Last modified: **Dec 2, 2018**</sup> -->

I wrote this post last year on my no-longer-maintained steemit blog, but I will re-post it here anyway. After I've been spending a lot of time working on containers and swarm clusters, when I read it again, it looks very basic but this can be the first of many posts about Docker and containerization.

Docker is an open-source platform for containerization. If you haven't heard what a container is, [docker.com](https://docker.com) defined it as follows:
"*A container image is a lightweight, stand-alone, executable package of a  piece of software that includes everything needed to run it: code,  runtime, system tools, system libraries, settings.*"

It's basically another way of virtualization on operating system level. It's more lightweight than a virtual machine because it uses the same kernel as the host OS. It creates an isolation for the virtual environment just by using linux namespaces and control groups. But more importantly, it is becoming the way we ship our software.

We will take a look at how to create a container from a docker configuration file, which is called a `Dockerfile`. Dockerfile simply includes the set of commands that are needed to run to prepare your custom container. These configurations are then run to generate a docker image, and those images can be run to create containers.

Many organizations have already official images for their products. They are available on Docker Store and can easily be inherited by your custom configurations.
In my example, we will create an Ubuntu server container for Python development purposes. In order to follow the next steps, you must install Docker on your computer by simply following the official instructions.

You just need to create a folder, create a text file under it and call it "Dockerfile". This is the default name docker uses to build an image when docker build command is executed within a folder. Now let's fill your Dockerfile. First, we will use the official Ubuntu base image. To inherit other images, you use the statement FROM.

```dockerfile
FROM ubuntu:latest
```

`:latest` is called a `tag` and it is used to check for a specific version defined in the image name, and you can also use other custom versions such as `:16.04`. These are image specific so each image may have different tags defined.

```dockerfile
RUN apt-get update && apt-get install -y \
apt-utils \
git \
emacs-nox \
vim \
curl \
wget \
psmisc \
tmux \
software-properties-common
```

Another statement you will use a lot is `RUN`. This is for basically what you execute in a shell, also will be run when your image is getting built. There's a convention for adding each command or package as a new line so that it's easier to read. As you can see I have custom packages that I prefer such as text editors, or terminal management tool like tmux. You can modify those however you find it fit for your needs. Now let's install Python 3 and its dependencies.

```dockerfile
RUN apt-get update && apt-get install -y \
python3 \
python3-pip \
python3-dev
```

I use pipenv for Python package dependency management, so I will install it using pip.

```dockerfile
RUN pip3 install --upgrade pip && \
hash -r pip pip3 && \
pip3 install pipenv
ENV SHELL /bin/bash
```

Pipenv requires SHELL environment variable so as you can guess, we define environment variables by ENV for our container. Now we will also set default locale for our container and our Dockerfile will be ready to be built.

```dockerfile
RUN apt-get update && apt-get install -y locales && rm -rf /var/lib/apt/lists/* \
&& localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG en_US.utf8
```

This part specifically was taken from official Postgres image configuration, which is also recommended by docker.com itself.

Now we are ready to build our image. Simply go to the parent folder of Dockerfile in terminal and run the following command.

```bash
docker build -t myimage:v0.0.1 .
```

Beware of the `.` in the end, which is the path for your Dockerfile. This command will download inherited images from Docker Store if not available locally and execute the commands to get your image ready. `-t` flag is used to tag your image in `name:tag` format. You can use any custom format of your choice for tagging your images. Once complete, you can see your local docker images by the following command.

```bash
docker image ls
```

Now you are ready to run your first container.

```bash
docker run -it --name mycontainer myimage:v0.0.1
```

This is the very basic command to start a container with a defined image. It will instantly create a container from your image and attach you to it's shell. You will see something on your terminal like this:

```bash
$ docker run -it --name mycontainer myimage:v0.0.1
root@9ec80324ebeb:/# 
```

This shows that you are logged in the container with ID `9ec80324ebeb` as `root` user. To detach from a container, you can use keyboard shortcut `Ctrl-p Ctrl-q`. Be careful, if you type `exit` or use `Ctrl-d` it does not just detach from the container but also stops it. To list running containers simply run the following command:

```bash
docker ps
```

To include exited containers, you can run `docker ps -a`. If you accidentally exit a container, you can start it by:

```bash
docker start mycontainer
```

If you want to attach to a running container, you can do it with:

```bash
docker attach mycontainer
```

And to stop a container when you are not attached to it:

```bash
docker stop mycontainer
```

This is the very basic of how to start a container using a Dockerfile. If you have any questions, I am more than happy to help.
