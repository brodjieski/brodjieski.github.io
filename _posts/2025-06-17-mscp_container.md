---
layout: post
title: "Using Containers to run MSCP"
date: 2025-06-17 00:00:00
published: true
hidden: true
categories: [MSCP]
tags: [mscp, docker, container]
---

# macOS and Containers

Following the announcment of macOS 26 at WWDC this year, a friend and colleague shared a link to a WWDC session on Containerization, which immediately caught my attention. During that session, we were introduced to a project from Apple that leveraged some new features in macOS 26 Beta. From the github page, [container](https://github.com/apple/container) is a tool that you can use to create and run Linux containers as lightweight virtual machines on your Mac. It's written in Swift, and optimized for Apple silicon. 

One of first things I thought about is how can we leverage this to make running the macOS Security Compliance Project (MSCP) even easier for Mac admins who want to build their compliance using the command line. 

One challenge of the MSCP has always been the dependencies required to run the scripts in the project. macOS does not include everything that is needed for the project. With `python` and `ruby` being two of the core components. The project has tried to leverage the pieces that Apple includes in either the shipped OS, or by way of the Xcode Command Line Tools, which give admins tools like `git` and `python`. The issue is that the tools that Apple provides are not always up to date. For example, `ruby` is included in macOS build, but is running version 2.6.10p210, which was released April 12... of 2022! While these versions have been functional for us over the years, we wanted to be able to modernize some of our tools which meant we need to be running newer versions of them.

One approach is that we provide the instructions to download and install all of the tools needed to run the project. While this is fine and most admins are fully capable of getting their environment setup, what if there was an easier way for us to have everything we needed with much less overhead. Something were all of the dependencies were included in a easy to deploy package... a container of sorts.

Containers are certainly not a new concept. Services like Docker and Podman have been available for quite some time on macOS. With macOS 26 Beta, Apple has brought the concept a bit closer to home. Let's see how we can build an MSCP container to generate some compliance documentation.

>While all of this was made available after WWDC and the announcment of macOS Tahoe, everything here, including the `container` application was tested and does work on macOS Sequoia. This does not require running macOS 26 Beta. It does require to be run directly on an Apple Silicon system and cannot be run in a virtual machine (VM).
{: .prompt-info }

## TL;DR
Download and install the latest signed installer package for `container` from Apple's [GitHub release page](https://github.com/apple/container/releases). 

```shell
container system start

container run -it ghcr.io/brodjieski/mscp:latest
```
For those who want to see the process in more detail, keep reading.

## Building the MSCP container
There are just a couple of steps needed to create a Linux container running the MSCP natively on macOS.

Download and install the latest signed installer package for `container` from Apple's [GitHub release page](https://github.com/apple/container/releases). 

Clone the macOS Security Compliance Project from NIST's GitHub page to your Desktop folder. 

```shell
git clone https://github.com/usnistgov/macos_security ~/Desktop/
```
>`git` requires the Xcode Command Line Tools
{: .prompt-info }


Create a `Dockerfile` file and place it in `~/Desktop/macos_security` with the following contents

```
FROM alpine:latest

# Install required packages
RUN apk update && apk add --no-cache \
  python3 \
  ruby \
  git \
  py3-pip \
  ruby-dev \
  build-base

# Set working directory
WORKDIR /mscp

# Install Python dependencies
COPY requirements.txt .
RUN pip install --break-system-packages --no-cache-dir -r requirements.txt

# Install Ruby dependencies
COPY Gemfile ./
RUN gem install bundler && bundle install

# Copy MSCP code
COPY . .

# Run a shell when container starts
CMD ["sh"]
``` 

This file tells the container engine how to build the container image and the necessary components. 

### Build the container image

In order for `container` to manage and interact with images and containers, the system service need to be started.

```shell
container system start
```

The first time you start the container system, it will install a base container filesystem and prompt you to install a default kernel. It provides a recommended default kernel for you to accept. 

Once the system has been started, you can now build the container from the `Dockerfile`.

```shell
container build -f ~/Desktop/macos_security/Dockerfile -t mscp ~/Desktop/macos_security/
```

Voila! You have successfully built an MSCP container tagged `mscp:latest` that contains everything needed to build compliance documents from MSCP.

## Using the MSCP container

Now that we have the container built, we can run the image with an interactive terminal. This will give us command line access to everything we need in the project.

```shell
container run -it mscp:latest
```

The `-it` flags tells `container` to run an interactive tty, which brings you to the shell within the container. You will notice that your command line prompt now reads:

```shell
/mscp # 
```

You can now run any of the MSCP scripts from this command line.

```shell
/mscp # script/generate_baseline.py -l

...

```

To exit the `mscp` shell, just type `exit` at the command prompt.

## Building compliance documents and saving them outside of the container

Once a container is created, the contents typically remain as-is and data that gets modified while inside the container will be erased when the container stops running. So how to we keep our generated compliance documents? When you run `generate_guidance.py` all of the output is sent to the `./build` folder, which resided inside the container and will be lost when exiting.

With the `--volume` option of `container run`, you can share data between the host system and one or more containers, and you can persist data across multiple container runs. The volume option allows you to mount a folder on your host to a filesystem path in the container.

To share the MSCP `./build` folder from the container with a `mscp` folder on your Desktop, run the following command:

```shell
container run -it --volume ~/Desktop/mscp:/mscp/build mscp:latest
```
>The host folder `~/Desktop/mscp` must exist
{: .prompt-warning }

Now when you run any of the generate scripts from within the container, the resulting files will be made available on your Desktop.

You can do multiple volumes as well. Let's say you also want to share your `custom` items used by the project:

```shell
container run -it --volume ~/Desktop/mscp:/mscp/build --volume ~/Desktop/mscp/custom:/mscp/custom mscp:latest
```

Now any customizations made will be persistant across container runs.

## Using an hosted container

For those who don't want to go through all of the process of building and maintaining their own container, you can always run a container that has been built and hosted just like any Docker container. An example can be found in my GitHub packages, and can be kicked off by telling `container` to use a hosted container, rather than a local.

```shell
container run -it --volume ~/Desktop/mscp:/mscp/build --volume ~/Desktop/mscp/custom:/mscp/custom ghcr.io/brodjieski/mscp:latest
```

Podman also works

```shell
podman run -it --volume /Users/<username>/Desktop/mscp:/mscp/build --volume /Users/<username>/Desktop/mscp/custom:/mscp/custom ghcr.io/brodjieski/mscp:latest
```

Using Docker has a similar command, although it requires full path names and commas in the `--volume` command

```shell
docker run -it --volume /Users/<username>/Desktop/mscp,/mscp/build --volume /Users/<username>/Desktop/mscp/custom,/mscp/custom ghcr.io/brodjieski/mscp:latest
```

# In Conclusion
Containers are a great way to deploy applications and solutions that can run natively in a Linux environment to include all of the required dependencies and software needed to use the application as expected. This use case is an example of how MSCP can be used on nearly any platform that can run containers... so now even your Windows admins have a means to generate macOS compliance documentation!