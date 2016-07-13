.. title: uWSGI, send_file and Python 3.5
.. slug: uwsgi-send_file-and-python35
.. date: 2016-07-05 20:39:29 UTC+02:00
.. tags: python,flask,uwsgi
.. category: python
.. link: 
.. description: 
.. type: text

I have a Flask app that returns an in-memory bytes buffer (`io.Bytesio
<https://docs.python.org/3/library/io.html#io.BytesIO>`_) using Flask `send_file
<http://flask.pocoo.org/docs/0.11/api/#flask.send_file>`_ function.

The app is deployed using uWSGI_ behind Nginx.
This was working fine with Python 3.4.

When I updated Python to 3.5, I got the following exception when trying to download a file::

    io.UnsupportedOperation: fileno

    During handling of the above exception, another exception occurred:

    Traceback (most recent call last):
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/flask/app.py", line 1817, in wsgi_app
        response = self.full_dispatch_request()
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/flask/app.py", line 1477, in full_dispatch_request
        rv = self.handle_user_exception(e)
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/flask/app.py", line 1381, in handle_user_exception
        reraise(exc_type, exc_value, tb)
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/flask/_compat.py", line 33, in reraise
        raise value
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/flask/app.py", line 1475, in full_dispatch_request
        rv = self.dispatch_request()
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/flask/app.py", line 1461, in dispatch_request
        return self.view_functions[rule.endpoint](**req.view_args)
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/flask_login.py", line 758, in decorated_view
        return func(*args, **kwargs)
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/flask_security/decorators.py", line 194, in decorated_view
        return fn(*args, **kwargs)
      File "/webapps/bowser/bowser/app/bext/views.py", line 116, in download
        as_attachment=True)
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/flask/helpers.py", line 523, in send_file
        data = wrap_file(request.environ, file)
      File "/webapps/bowser/miniconda3/envs/bowser/lib/python3.5/site-packages/werkzeug/wsgi.py", line 726, in wrap_file
        return environ.get('wsgi.file_wrapper', FileWrapper)(file, buffer_size)
    SystemError: <built-in function uwsgi_sendfile> returned a result with an error set


I quickly found the following `post <http://lists.unbit.it/pipermail/uwsgi/2015-September/008186.html>`_ with the same exception, but no answer...
A little more googling brought me to this github issue: `In python3, uwsgi fails to respond a
stream from BytesIO object <https://github.com/unbit/uwsgi/issues/1126>`_

As described, you should run uwsgi with the `--wsgi-disable-file-wrapper` flag to avoid this problem.
As with all command line options, you can add the following entry in your
uwsgi.ini file::

    wsgi-disable-file-wrapper = true


Note that uWSGI_ 2.0.12 is required.

When searching in uWSGI_ documentation, I only found one match in `uWSGI 2.0.12 release notes
<http://uwsgi-docs.readthedocs.io/en/latest/Changelog-2.0.12.html?highlight=wsgi-disable-file-wrapper>`_.

A problem/option that should be better documented. Probably a pull request to open :-)

UPDATE (2016-07-13): pull request `merged <https://github.com/unbit/uwsgi-docs/pull/317>`_

.. _uWSGI: http://uwsgi-docs.readthedocs.io/en/latest/
