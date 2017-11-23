.. title: Running your application over HTTPS with traefik
.. slug: running-your-application-over-https-with-traefik
.. date: 2017-06-08 21:49:46 UTC+02:00
.. tags: python,flask,traefik,https
.. category: python
.. link:
.. description:
.. type: text

I just read another very clear article from Miguel Grinberg about `Running Your Flask
Application Over HTTPS
<https://blog.miguelgrinberg.com/post/running-your-flask-application-over-https>`_.

As the title suggests, it describes different ways to run a flask application over HTTPS.
I have been using flask_ for quite some time, but I didn't even know about
the `ssl_context` argument. You should definitively check his article!

Using nginx_ as a reverse proxy with a self-signed certificate or `Let’s
Encrypt <https://letsencrypt.org>`_ are two options I have been using in the
past.

If your app is available on the internet, you should definitively use
Let's Encrypt. But if your app is only supposed to be used internally
on a private network, a self-signed certificate is an option.

Traefik
-------

I now often use docker_ to deploy my applications.
I was looking for a way to automatically configure Let's Encrypt.
I initially found `nginx-proxy <https://github.com/jwilder/nginx-proxy>`_
and `docker-letsencrypt-nginx-proxy-companion
<https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion>`_. This
was interesting but wasn't that straight forward to setup.

I then discovered traefik_: "a modern HTTP reverse proxy
and load balancer made to deploy microservices with ease". And that's
really the case! I've used it to deploy several applications and I
was impressed. It's written in go, so single binary. There is also a tiny docker
image that makes it easy to deploy. It includes Let's Encrypt support (with automatic renewal),
websocket support (no specific setup required)... And many other features.

Here is a `traefik.toml` configuration example::

    defaultEntryPoints = ["http", "https"]

    [web]
    # Port for the status page
    address = ":8080"

    # Entrypoints, http and https
    [entryPoints]
      # http should be redirected to https
      [entryPoints.http]
      address = ":80"
    	[entryPoints.http.redirect]
    	entryPoint = "https"
      # https is the default
      [entryPoints.https]
      address = ":443"
    	[entryPoints.https.tls]

    # Enable ACME (Let's Encrypt): automatic SSL
    [acme]
    # Email address used for registration
    email = "test@traefik.io"
    storageFile = "/etc/traefik/acme/acme.json"
    entryPoint = "https"
    onDemand = false
    OnHostRule = true

    # Enable Docker configuration backend
    [docker]
    endpoint = "unix:///var/run/docker.sock"
    domain = "example.com"
    watch = true
    exposedbydefault = false


With this simple configuration, you get:

- HTTP redirect on HTTPS
- Let's Encrypt support
- Docker backend support

A simple example
----------------

I created a dummy example just to show how to run a flask application over
HTTPS with traefik and Let's Encrypt.
Note that traefik is made to dynamically discover backends. So you usually
don't run it with your app in the same docker-compose.yml file. It usually
runs separately. But to make it easier, I put both in the same file::

    version: '2'
    services:
      flask:
        build: ./flask
        image: flask
        command: uwsgi --http-socket 0.0.0.0:5000 --wsgi-file app.py --callable app
        labels:
          - "traefik.enable=true"
          - "traefik.backend=flask"
          - "traefik.frontend.rule=${TRAEFIK_FRONTEND_RULE}"
      traefik:
        image: traefik
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock:ro
          - ./traefik/traefik.toml:/etc/traefik/traefik.toml:ro
          - ./traefik/acme:/etc/traefik/acme
        ports:
         - "80:80"
         - "443:443"
         - "8080:8080"

Traefik requires access to the docker socket to listen for changes in the
backends. It can thus automatically discover when you start and stop
containers. You can ovverride default behaviour by using labels in your
container.

Supposing you own the myhost.example.com domain and have access to ports 80 and 443
(you can setup port forwarding if you run that on your machine behind a
router at home), you can run::

    $ git clone https://github.com/beenje/flask_traefik_letsencrypt.git
    $ cd flask_traefik_letsencrypt
    $ export TRAEFIK_FRONTEND_RULE=Host:myhost.example.com
    $ docker-compose up

Voilà! Our flask app is available over HTTPS with a real SSL certificate!

.. image:: /images/flask_traefik/hello_world.png

Traefik discovered the flask docker container and requested a certificate for our domain.
All that automatically!

Traefik even comes with a nice dashboard:

.. image:: /images/flask_traefik/traefik_dashboard.png

With this simple configuration, `Qualys SSL Labs <https://www.ssllabs.com/index.html>`_
gave me an A rating :-)

.. image:: /images/flask_traefik/traefik_ssl_report.png

Not as good as the A+ for Miguel's site, but not that bad!
Especially considering there isn't any specific SSL setup.

A more realistic deployment
---------------------------

As I already mentioned, traefik is made to automatically discover backends (docker containers in my case).
So you usually run it by itself.

Here is an example how it can be deployed using Ansible_::

    ---
    - name: create traefik directories
      file:
        path: /etc/traefik/acme
        state: directory
        owner: root
        group: root
        mode: 0755

    - name: create traefik.toml
      template:
        src: traefik.toml.j2
        dest: /etc/traefik/traefik.toml
        owner: root
        group: root
        mode: 0644
      notify:
        - restart traefik

    - name: create traefik network
      docker_network:
        name: "{{traefik_network}}"
        state: present

    - name: launch traefik container with letsencrypt support
      docker_container:
        name: traefik_proxy
        image: "traefik:{{traefik_version}}"
        state: started
        restart_policy: always
        ports:
          - "80:80"
          - "443:443"
          - "{{traefik_dashboard_port}}:8080"
        volumes:
          - /etc/traefik/traefik.toml:/etc/traefik/traefik.toml:ro
          - /etc/traefik/acme:/etc/traefik/acme:rw
          - /var/run/docker.sock:/var/run/docker.sock:ro
        # purge networks so that the container is only part of
        # {{traefik_network}} (and not the default bridge network)
        purge_networks: yes
        networks:
          - name: "{{traefik_network}}"

    - name: force all notified handlers to run
      meta: flush_handlers

Nothing strange here. It's quite similar to what we had in our docker-compose.yml file.
We created a specific `traefik_network`. Our docker containers will have to be on that same network.

Here is how we could deploy a flask application on the same server using another ansible role::

    - name: launch flask container
      docker_container:
        name: flask
        image: flask
        command: uwsgi --http-socket 0.0.0.0:5000 --wsgi-file app.py --callable app
        state: started
        restart_policy: always
        purge_networks: yes
        networks:
          - name: "{{traefik_network}}"
        labels:
          traefik.enable: "true"
          traefik.backend: "flask"
          traefik.frontend.rule: "Host:myhost.example.com"
          traefik.port: "5000"

We make sure the container is on the same network as the traefik proxy.
Note that the `traefik.port` label is only required if the container exposes multiple ports.
It's thus not needed in our example.

That's basically it. As you can see, docker_ and Ansible_ make the deployment easy.
And traefik_ takes care of the Let's Encrypt certificate.

Conclusion
----------

Traefik comes with many other features and is well `documented <https://docs.traefik.io>`_.
You should check `this Docker example <https://docs.traefik.io/#docker>`_ that demonstrates load-balancing.
Really cool.

If you use docker_, you should really give traefik_ a try!

.. _flask: http://flask.pocoo.org
.. _nginx: https://nginx.org/en/
.. _docker: https://www.docker.com
.. _traefik: https://traefik.io
.. _Ansible: http://docs.ansible.com/ansible/
