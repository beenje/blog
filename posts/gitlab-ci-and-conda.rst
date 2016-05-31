.. title: GitLab CI and conda
.. slug: gitlab-ci-and-conda
.. date: 2016-05-31 16:48:23 UTC+02:00
.. tags: python,conda,gitlab,ci,git
.. category: python
.. link:
.. description:
.. type: text

I setup GitLab to host several projects at work and I have been quite
pleased with it. I read that setting GitLab CI for test and deployment was
easy so I decided to try it to automatically run the test suite and the
sphinx documentation.

I found the official `documentation
<http://docs.gitlab.com/ce/ci/quick_start/README.html>`_ to be quite good
to setup a runner so I won't go into details here. I chose the `Docker
executor
<https://gitlab.com/gitlab-org/gitlab-ci-multi-runner/blob/master/docs/executors/docker.md>`_.

Here is my first `.gitlab-ci.yml` test::

    image: python:3.4

    before_script:
      - pip install -r requirements.txt

    tests:
      stage: test
      script:
        - python -m unittest discover -v

Success, it works! Nice. But... 8 minutes 33 seconds build time for a test suite that
runs in less than 1 second... that's a bit long.

Let's try using some caching to avoid having to download all the pip
requirements every time. After googling, I found this `post
<https://fleschenberg.net/gitlab-pip-cache/>`_ explaining that the cache
path must be inside the build directory::

    image: python:3.4

    before_script:
      - export PIP_CACHE_DIR="pip-cache"
      - pip install -r requirements.txt

    cache:
      paths:
        - pip-cache

    tests:
      stage: test
      script:
        - python -m unittest discover -v

With the pip cache, the build time went down to about 6 minutes. A bit
better, but far from acceptable.

Of course I knew the problem was not the download, but the
installation of the pip requirements. I use pandas_
which explains why it takes a while to compile.

So how do you install pandas_ easily? With conda_ of course!
There are even some nice `docker images
<https://github.com/ContinuumIO/docker-images>`_  created by Continuum Analytics ready to be used.

So let's try again::

    image: continuumio/miniconda3:latest

    before_script:
      - conda env create -f environment.yml
      - source activate koopa

    tests:
      stage: test
      script:
        - python -m unittest discover -v

Build time: 2 minutes 55 seconds. Nice but we need some cache to avoid
downloading all the packages everytime.
The first problem is that the cache path has to be in the build directory.
Conda packages are saved in `/opt/conda/pkgs` by default. A solution is to
replace that directory with a link to a local directory. It works but the
problem is that Gitlab makes a compressed archive to save and restore the
cache which takes quite some time in this case...

How to get a fast cache? Let's use a docker volume!
I modified my `/etc/gitlab-runner/config.toml` to add two volumes::

    [runners.docker]
      tls_verify = false
      image = "continuumio/miniconda3:latest"
      privileged = false
      disable_cache = false
      volumes = ["/cache", "/opt/cache/conda/pkgs:/opt/conda/pkgs:rw", "/opt/cache/pip:/opt/cache/pip:rw"]

One volume for conda_ packages and one for `pip`.
My new `.gitlab-ci.yml`::

    image: continuumio/miniconda3:latest

    before_script:
      - export PIP_CACHE_DIR="/opt/cache/pip"
      - conda env create -f environment.yml
      - source activate koopa

    tests:
      stage: test
      script:
        - python -m unittest discover -v

The build time is about 10 seconds!

Just a few days after my tests, GitLab announced `GitLab Container
Registry
<https://about.gitlab.com/2016/05/23/gitlab-container-registry/>`_.
I already thought about building my own docker image and this new feature
would make it even easier than before. But I would have to remember to update
my image if I change my requirements. Which I don't have to think about with the
current solution.


.. _pandas: http://pandas.pydata.org
.. _conda: http://conda.pydata.org/docs/
