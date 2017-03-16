.. title: Dockerfile anti-patterns and best practices
.. slug: dockerfile-anti-patterns-and-best-practices
.. date: 2017-03-16 22:10:49 UTC+01:00
.. tags: docker
.. category: docker
.. link:
.. description:
.. type: text

I've been using Docker_ for some time now.
There is already a lot of documentation available online but I recently
saw the same "anti-patterns" several times, so I thought it was worth writing a post about
it.

I won't repeat all the `Best practices for writing Dockerfiles
<https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/>`_ here.
You should definitively read that page.

I want to emphasize some things that took me some time to understand.

Avoid invalidating the cache
----------------------------

Let's take a simple example with a Python application::

    FROM python:3.6

    COPY . /app
    WORKDIR /app

    RUN pip install -r requirements.txt

    ENTRYPOINT ["python"]
    CMD ["ap.py"]

It's actually an example I have seen several times online.
This looks fine, right?

The problem is that the *COPY . /app* command will invalidate the cache as
soon as any file in the current directory is updated.
Let's say you just change the *README* file and run *docker build* again.
Docker will have to re-install all the requirements because the
*RUN pip* command is run after the *COPY* that invalidated the cache.

The requirements should only be re-installed if the *requirements.txt*
file changes::

    FROM python:3.6

    WORKDIR /app

    COPY requirements.txt /app/requirements.txt
    RUN pip install -r requirements.txt

    COPY . /app

    ENTRYPOINT ["python"]
    CMD ["ap.py"]

With this Dockerfile, the *RUN pip* command will only be re-run when the
*requirements.txt* file changes. It will use the cache otherwise.

This is much more efficient and will save you quite some time if you have
many requirements to install.


Minimize the number of layers
-----------------------------

What does that really mean?

Each Docker_ image references a list of read-only layers that represent
filesystem differences. Every command in your Dockerfile will create a new
layer.

Let's use the following Dockerfile::

    FROM centos:7

    RUN yum update -y
    RUN yum install -y sudo
    RUN yum install -y git
    RUN yum clean all

Build the docker image and check the layers created with the *docker history* command::

    $ docker build -t centos-test .
    ...
    $ docker images
    REPOSITORY                       TAG                 IMAGE ID            CREATED              SIZE
    centos-test                      latest              1fae366a2613        About a minute ago   470 MB
    centos                           7                   98d35105a391        24 hours ago         193 MB
    $ docker history centos-test
    IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
    1fae366a2613        2 minutes ago       /bin/sh -c yum clean all                        1.67 MB
    999e7c7c0e14        2 minutes ago       /bin/sh -c yum install -y git                   133 MB
    c97b66528792        3 minutes ago       /bin/sh -c yum install -y sudo                  81 MB
    e0c7b450b7a8        3 minutes ago       /bin/sh -c yum update -y                        62.5 MB
    98d35105a391        24 hours ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0 B
    <missing>           24 hours ago        /bin/sh -c #(nop)  LABEL name=CentOS Base ...   0 B
    <missing>           24 hours ago        /bin/sh -c #(nop) ADD file:29f66b8b4bafd0f...   193 MB
    <missing>           6 months ago        /bin/sh -c #(nop)  MAINTAINER https://gith...   0 B

There are two problems with this Dockerfile:

1. We added too many layers for nothing.
2. The *yum clean all* command is meant to reduce the size of the image but it
   actually does the opposite by adding a new layer!

Let's check that by removing the latest command and running the build
again::

    FROM centos:7

    RUN yum update -y
    RUN yum install -y sudo
    RUN yum install -y git
    # RUN yum clean all

::

    $ docker build -t centos-test .
    ...
    $ docker images
    REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
    centos-test                      latest              999e7c7c0e14        11 minutes ago      469 MB
    centos                           7                   98d35105a391        24 hours ago        193 MB

The new image without the *yum clean all* command is indeed smaller than the previous image (1.67 MB smaller)!

If you want to remove files, it's important to do that in the same RUN command that created those files.
Otherwise there is no point.

Here is the proper way to do it::

    FROM centos:7

    RUN yum update -y \
      && yum install -y \
      sudo \
      git \
      && yum clean all

Let's build this new image::

    $ docker build -t centos-test .
    ...
    $ docker images
    REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
    centos-test                      latest              54a328ef7efd        21 seconds ago      265 MB
    centos                           7                   98d35105a391        24 hours ago        193 MB
    $ docker history centos-test
    IMAGE               CREATED              CREATED BY                                      SIZE                COMMENT
    54a328ef7efd        About a minute ago   /bin/sh -c yum update -y   && yum install ...   72.8 MB
    98d35105a391        24 hours ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0 B
    <missing>           24 hours ago         /bin/sh -c #(nop)  LABEL name=CentOS Base ...   0 B
    <missing>           24 hours ago         /bin/sh -c #(nop) ADD file:29f66b8b4bafd0f...   193 MB
    <missing>           6 months ago         /bin/sh -c #(nop)  MAINTAINER https://gith...   0 B

The new image is only 265 MB compared to the 470 MB of the original image.
There isn't much more to say :-)

If you want to know more about images and layers, you should read the
documentation: `Understand images, containers, and storage drivers
<https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/>`_.

Conclusion
----------

Avoid invalidating the cache:

- start your Dockerfile with commands that should not change often
- put commands that can often invalidate the cache (like COPY .) as
  late as possible
- only add the needed files (use a .dockerignore file)

Minimize the number of layers:

- put related commands in the same RUN instruction
- remove files in the same RUN command that created them


.. _Docker: https://www.docker.com
