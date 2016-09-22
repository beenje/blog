.. title: GitLab Container Registry and proxy
.. slug: gitlab-container-registry-and-proxy
.. date: 2016-09-21 22:10:06 UTC+02:00
.. tags: gitlab,ci,git,docker,synology
.. category: gitlab
.. link:
.. description:
.. type: text

GitLab on Synology
------------------

I installed GitLab CE on a Synology RackStation RS815+ at work.
It has an Intel Atom C2538 that allows to run Docker_ on the NAS.

Official GitLab Community Edition docker images are available on `Docker Hub <https://hub.docker.com/r/gitlab/gitlab-ce/>`_.
The documentation to use the image is quite clear and can be found `here <https://docs.gitlab.com/omnibus/docker/>`_.

The ports 80 and 443 are already used by nginx that comes with DSM_.
I wanted to access GitLab using HTTPS, so I disabled port 443 in nginx
configuration. To do that I had to modify the template
`/usr/syno/share/nginx/WWWService.mustache` and reboot the NAS:

.. code:: diff

    --- WWWService.mustache.org 2016-08-16 23:25:06.000000000 +0100
    +++ WWWService.mustache 2016-09-19 13:53:45.256735700 +0100
    @@ -1,8 +1,6 @@
     server {
         listen 80 default_server{{#reuseport}} reuseport{{/reuseport}};
         listen [::]:80 default_server{{#reuseport}} reuseport{{/reuseport}};
    -    listen 443 default_server ssl{{#reuseport}} reuseport{{/reuseport}};
    -    listen [::]:443 default_server ssl{{#reuseport}} reuseport{{/reuseport}};

         server_name _;

The port 22 is also already used by the ssh daemon so I decided to use
the port 2222. I created the directory `/volume1/docker/gitlab` to store
all GitLab data. Here are the required variables in the
`/volume1/docker/gitlab/config/gitlab.rb` config file::

    external_url "https://mygitlab.example.com"

    ## GitLab Shell settings for GitLab
    gitlab_rails['gitlab_shell_ssh_port'] = 2222

    nginx['enable'] = true
    nginx['redirect_http_to_https'] = true

And this is how I run the image::

    docker run --detach \
        --hostname mygitlab.example.com \
        --publish 443:443 --publish 8080:80 --publish 2222:22 \
        --name gitlab \
        --restart always \
        --volume /volume1/docker/gitlab/config:/etc/gitlab \
        --volume /volume1/docker/gitlab/logs:/var/log/gitlab \
        --volume /volume1/docker/gitlab/data:/var/opt/gitlab \
        gitlab/gitlab-ce:latest


This has been working fine. Since I heard about `GitLab Container Registry <https://about.gitlab.com/2016/05/23/gitlab-container-registry/>`_,
I've been wanted to give it a try.

GitLab Container Registry
-------------------------

To enable it, I just added to my `gitlab.rb` file the registry url::

    registry_external_url 'https://mygitlab.example.com:4567'

I use the existing GitLab domain and use the port 4567 for the registry.
The TLS certificate and key are in the default path, so no need to specify them.

So let's restart GitLab. Don't forget to publish the new port 4567!

::

    $ docker stop gitlab
    $ docker rm gitlab
    $ docker run --detach \
        --hostname mygitlab.example.com \
        --publish 443:443 --publish 8080:80 --publish 2222:22 \
        --publish 4567:4567 \
        --name gitlab \
        --restart always \
        --volume /volume1/docker/gitlab/config:/etc/gitlab \
        --volume /volume1/docker/gitlab/logs:/var/log/gitlab \
        --volume /volume1/docker/gitlab/data:/var/opt/gitlab \
        gitlab/gitlab-ce:latest


Easy! Let's test our new docker registry!

::

    $ docker login mygitlab.example.com:4567
    Username: user
    Password:
    Error response from daemon: Get https://mygitlab.example.com:4567/v1/users/: Service Unavailable

Hmm... Not super useful error...
I thought about publishing port 4567 in docker, so what is happening?
After looking through the logs, I found `/volume1/docker/gitlab/logs/nginx/gitlab_registry_access.logi`. It's empty...
Let's try curl::

    $ curl https://mygitlab.example.com:4567/v1/users/

    curl: (60) Peer certificate cannot be authenticated with known CA certificates
    More details here: http://curl.haxx.se/docs/sslcerts.html

    curl performs SSL certificate verification by default, using a "bundle"
     of Certificate Authority (CA) public keys (CA certs). If the default
     bundle file isn't adequate, you can specify an alternate file
     using the --cacert option.
    If this HTTPS server uses a certificate signed by a CA represented in
     the bundle, the certificate verification probably failed due to a
     problem with the certificate (it might be expired, or the name might
     not match the domain name in the URL).
    If you'd like to turn off curl's verification of the certificate, use
     the -k (or --insecure) option.


OK, I have a self-signed certificate. So let's try with `--insecure`::

    $ curl --insecure https://mygitlab.example.com:4567/v1/users/
    404 page not found

At least I get an entry in my log file::

    $ cd /volume1/docker/gitlab
    $ cat logs/nginx/gitlab_registry_access.log
    xxx.xx.x.x - - [21/Sep/2016:14:24:57 +0000] "GET /v1/users/ HTTP/1.1" 404 19 "-" "curl/7.43.0"

So, docker and nginx seem to be configured properly...
It looks like `docker login` is not even trying to access my host...

Let's try with a dummy host::

    $ docker login foo
    Username: user
    Password:
    Error response from daemon: Get https://mygitlab.example.com:4567/v1/users/: Service Unavailable

Same error!
Why is that? I can ping `mygitlab.example.com` and even access nginx on port 4567 (using curl)
inside the docker container...
My machine is on the same network. It can't be a proxy problem. Wait. Proxy?

That's when I remembered I had configured my docker daemon to use a proxy to access the internet!
I created the file `/etc/systemd/system/docker.service.d/http-proxy.conf` with::

    [Service]
    Environment="HTTP_PROXY=http://proxy.example.com:8080/"

Reading the `docker documentation <https://docs.docker.com/engine/admin/systemd/>`_, it's very clear:
**If you have internal Docker registries that you need to contact without proxying you can specify them via the NO_PROXY environment variable**

Let's add the NO_PROXY variable::

    [Service]
    Environment="HTTP_PROXY=http://proxy.example.com:8080/" "NO_PROXY=localhost,127.0.0.1,mygitlab.example.com"

Flush the changes and restart the docker daemon::

    $ sudo systemctl daemon-reload
    $ sudo systemctl restart docker


Now let's try to login again::

    $ docker login mygitlab.example.com:4567
    Username: user
    Password:
    Error response from daemon: Get https://mygitlab.example.com:4567/v1/users/: x509: certificate signed by unknown authority

This error is easy to fix (after googling). I have to add the self-signed certificate at the OS level.
On my Ubuntu machine::

    $ sudo cp mygitlab.example.com.crt /usr/local/share/ca-certificates/
    $ sudo update-ca-certificates
    $ sudo systemctl restart docker

    $ docker login mygitlab.example.com:4567
    Username: user
    Password: 
    Login Succeeded

Yes! :-)

I can now push docker images to my GitLab Container Registry!

Conclusion
----------

Setting GitLab Container Registry should have been easy but my proxy
settings made me lost quite some time... The proxy environment variables (HTTP_PROXY, NO_PROXY...)
are not taken into account by the docker commands. The docker daemon has to be configured
specifically. Something to remember!

Note that this was with docker 1.11.2. When trying the same command on my Mac with docker 1.12.1, I got a nicer error message::

    $ docker --version
    Docker version 1.12.1, build 6f9534c
    $ docker login foo
    Username: user
    Password:
    Error response from daemon: Get https://foo/v1/users/: dial tcp: lookup foo on xxx.xxx.xx.x:53: no such host


.. _Docker: https://www.docker.com
.. _DSM: https://www.synology.com/en-global/dsm/6.0
