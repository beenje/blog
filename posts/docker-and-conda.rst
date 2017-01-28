.. title: Docker and conda
.. slug: docker-and-conda
.. date: 2017-01-28 23:32:56 UTC+01:00
.. tags: docker,conda,python
.. category: docker
.. link: 
.. description: 
.. type: text

I just read a blog post about `Using Docker with Conda Environments
<http://fmgdata.kinja.com/using-docker-with-conda-environments-1790901398>`_.
I do things slightly differently so I thought I would share an example of
Dockerfile I use:

::

    FROM continuumio/miniconda3:latest

    # Install extra packages if required
    RUN apt-get update && apt-get install -y \
        xxxxxx \
        && rm -rf /var/lib/apt/lists/*

    # Add the user that will run the app (no need to run as root)
    RUN groupadd -r myuser && useradd -r -g myuser myuser

    WORKDIR /app

    # Install myapp requirements
    COPY environment.yml /app/environment.yml
    RUN conda config --add channels conda-forge \
        && conda env create -n myapp -f environment.yml \
        && rm -rf /opt/conda/pkgs/*

    # Install myapp
    COPY . /app/
    RUN chown -R myuser:myuser /app/*
    
    # activate the myapp environment
    ENV PATH /opt/conda/envs/myapp/bin:$PATH


I don't run `source activate myapp` but just use `ENV` to update the `PATH`
variable. There is only one environment in the docker image. No need for the extra
checks done by the activate script.

With this Dockerfile, any command will be run in the `myapp`
environment.

Just a few additional notes:

1. Be sure to only copy the file `environment.yml` before to copy the full
   current directory. Otherwise any change in the directory would
   invalidate the docker cache.
   We only want to re-create the conda environment if `environment.yml`
   changes.
2. I always add the `conda-forge channel
   <https://conda-forge.github.io>`_. 
   Check this `post
   <https://www.continuum.io/blog/developer-blog/community-conda-forge>`_
   if you haven't heard of it yet.
3. I clean some cache (*/var/lib/apt/lists/* and */opt/conda/pkgs/*) to
   make the image a bit smaller.

I switched from virtualenv to conda_ a while ago and I really enjoy it.
A big thanks to `Continuum Analytics <https://www.continuum.io>`_!

.. _conda: https://conda.io
.. _virtualenv: https://virtualenv.pypa.io/en/stable/
.. _Docker: https://www.docker.com
