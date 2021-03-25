# How to make a non-root based container

## Introduction/Scope

:wave: Hi! So you've realized that you're container is run as `root`. Don't worry
a lot of containers out there do that. But if you're moving towards the operator
and OpenShift ecosystem, you've probably realized you'll need refactor some code.

This tutorial should be a simple framework to help you grasp and hopefully get you
on the path to success. 

> Note: Every application has different requirements and this isn't to teach you
> how to do _yours_. This is just a tutorial to get you in the correct head space
> so you can start working towards it.

## Steps

The first thing you need to do is understand how permissions work in containers.
These next two `Dockerfiles` should help you understand how you can leverage
the users.

First lets just make a container that runs as `root`:
```Dockerfile
FROM registry.access.redhat.com/ubi8/ubi-minimal

USER root

ENTRYPOINT ["whoami"]
```

Now lets create one that runs as `app`:

```Dockerfile
FROM registry.access.redhat.com/ubi8/ubi-minimal

RUN microdnf --enablerepo=ubi-8-baseos install shadow-utils && \
    microdnf update -y

RUN useradd --create-home app
WORKDIR /home/app
USER app

ENTRYPOINT ["whoami"]
```

Go ahead and build both the containers, and then run them, you should see `root`
and `app` as the outputs.

Wonderful! So as you can see this now two different containers with two different
privliges. 

Now that you see you can just create a user, (`app`) you should be able to take your
application and the configuration changes you need after the `USER` line. If you need
to install more depencies or other libaries be sure to do it _before_ you switching
to `app`.

The best way is to experiment, but when you get your application to run as the non-root
user, you're one step closer to getting your application running on OpenShift!
