.. _web:

Running under a Web Server
==========================

Running Pootle under a proper web server will improve performance, give you more
flexibility, and might be better for security. It is strongly recommended to
run Pootle under Apache, Nginx, or a similar web server.


.. _apache:

Running under Apache
--------------------

You can use Apache either as a reverse proxy or straight with mod_wsgi.


.. _apache#reverse_proxy:

Proxying with Apache
^^^^^^^^^^^^^^^^^^^^

If you want to reverse proxy through Apache, you will need to have `mod_proxy
<https://httpd.apache.org/docs/current/mod/mod_proxy.html>`_ installed for
forwarding requests and configure it accordingly.

.. code-block:: apache

    ProxyPass / http://localhost:8000/
    ProxyPassReverse / http://localhost:8000/


.. _apache#mod_wsgi:

Apache with mod_wsgi
^^^^^^^^^^^^^^^^^^^^

Make sure to review your global Apache settings (something like
*/etc/apache2/apache2.conf* for Debian-based or */etc/httpd/conf/httpd.conf* for
RedHat-based) for the server-pool settings. The default settings provided by
Apache are too high for running a web application like Pootle. The ideal
settings depend heavily on your hardware andthe number of users you expect to
have. A moderate server with 1GB memory might set ``MaxClients`` to something
like ``20``, for example.

Make sure Apache has read access to all of Pootle's files and write access to
the :setting:`PODIRECTORY` directory.

Preparing Apache
""""""""""""""""

If you have only one virtual host on your server, it is possible to edit only
the main Apache configuration file (mentioned above). If you have several hosts,
it is recommended to separate them into different files. These are typically
stored in the *sites-available* subdirectory of where your Apache configuration
file is stored and symbolically linked to the *sites-enabled* subdirectory. If
these do not exist, you can create them as follows:

.. code-block:: shell

    cd /path/to/apache/config/file/
    mkdir sites-enabled sites-available
    touch sites-available/pootle
    ln -s ../sites-available/pootle sites-enabled/pootle

Then add the following to your Apache configuration file if it isn't already
there:

.. code-block:: apache

    # Include the virtual host configurations:
    Include sites-enabled/

Accessing pootle can be done either insecurely through HTTP or securely through
HTTPS, however accessing the Admin section **requires** a secure connection. If
you don't have any SSL certificates, here are some `instructions for creating
self-signed certificates
<http://www.linux.com/learn/tutorials/392099:creating-self-signed-ssl-certificates-for-apache-on-linux>`_
on Linux.

Updating Apache conf files
""""""""""""""""""""""""""

You should then create two VirtualHost directives in the pootle file you just
created, one for secure connections (on port 443) and another for insecure
connections (on port 80). Note, you can change the ports to whatever you like,
but if you change them, you will have to specify the port number in the URL when
accessing Pootle (e.g. https://127.0.0.1:8000).

Here is a sample pootle configuration file over a secure connection. For
insecure connections just change the port from 80 to 443 and remove the lines
about SSL:

.. code-block:: apache

    <VirtualHost *:443>
    # Set the environment variable containing the path to your main pootle.conf
    # file
    SetEnv POOTLE_SETTINGS /var/www/pootle/pootle.conf

    # Important: Comment these lines out if you are not using HTTPS
    SSLEngine on
    SSLCertificateFile /path/to/ssl.crt
    SSLCertificateKeyFile /path/to/ssl.key
    # Uncomment this if you have a certificate chain file
    # SSLCertificateChainFile /path/to/ssl.ca

    # Point to the WSGI loader script. If you want to have pootle accessed by
    # going to the http://SERVER_NAME/pootle subdirectory instead of the
    # web root, change / to /pootle (and also update the Alias options below).
    WSGIScriptAlias / /var/www/pootle/wsgi.py

    # The following two optional lines enables "daemon mode" which limits the
    # number of processes and therefore also keeps memory use more predictable
    #
    # Note: the python-path needs to be set to the apps subdirectory of where
    # the pootle source files are stored. These may be under
    # /var/www/pootle/src/apps or something like
    # /usr/local/lib/python2.6/dist-packages/pootle/apps
    WSGIDaemonProcess pootle processes=2 threads=3 stack-size=1048576 maximum-requests=5000 inactivity-timeout=900 display-name=%{GROUP} python-path=/path/to/pootle/apps
    WSGIProcessGroup pootle

    # Directly serve static files like css and images, no need to go through
    # mod_wsgi and django
    Alias /assets /var/www/pootle/assets
    <Directory /var/www/Pootle/assets>
        Order deny,allow
        Allow from all
    </Directory>

    # Allow downloading translation files directly
    Alias /export /var/www/pootle/po
    <Directory /var/www/pootle/po>
        Order deny,allow
        Allow from all
    </Directory>

You can find more information in the `Django docs about Apache and
mod_wsgi <https://docs.djangoproject.com/en/dev/howto/deployment/wsgi/modwsgi/>`_ 
and `more information about mod_wsgi configuration directives  
<http://code.google.com/p/modwsgi/wiki/ConfigurationDirectives#WSGIDaemonProcess>`_.


.. _apache#.htaccess:

.htaccess
"""""""""

If you do not have access to the main Apache configuration, you should still be
able to configure things correctly using the *.htaccess* file.

`More information
<http://code.google.com/p/modwsgi/wiki/ConfigurationGuidelines>`_ on
configuring *mod_wsgi* (including *.htaccess*)


.. _nginx:

Running under Nginx
-------------------

Running Pootle under a web server such as Nginx will improve performance. For
more information about Nginx and WSGI, visit `Nginx's WSGI page
<http://wiki.nginx.org/NginxNgxWSGIModule>`_

A Pootle server is made up of static and dynamic content. By default Pootle
serves all content, and for low-latency purposes it is better to get other
webserver to serve the content that does not change, the static content. It is
just the issue of low latency and making the translation experience more
interactive that calls you to proxy through Nginx.  The following steps show you
how to setup Pootle to proxy through Nginx.


.. _nginx#proxy:

Proxying with Nginx
^^^^^^^^^^^^^^^^^^^

The default Pootle server runs at port 8000 and for convenience and simplicity
does ugly things such as serving static files — you should definitely avoid that
in production environments.

By proxying Pootle through nginx, the web server will serve all the static media
and the dynamic content will be produced by the app server.

.. code-block:: nginx

   server {
      listen  80;
      server_name  pootle.example.com;

      access_log /path/to/pootle/logs/nginx-access.log;

      charset utf-8;

      location /assets {
          alias /path/to/pootle/env/lib/python2.6/site-packages/pootle/assets/;
          expires 14d;
          access_log off;
      }

      location / {
        proxy_pass         http://localhost:8000;
        proxy_redirect     off;

        proxy_set_header   Host             $host;
        proxy_set_header   X-Real-IP        $remote_addr;
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
      }
    }


.. _nginx#proxy_fastcgi:

Proxying with Nginx (FastCGI)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Run Pootle as a FastCGI application::

    $ pootle runfcgi host=127.0.0.1 port=8080

There are more possible parameters available. See::

    $ pootle help runfcgi

And add the following lines to your Nginx config file:

.. code-block:: nginx

   server {
      listen  80;  # port and optionally hostname where nginx listens
      server_name  example.com translate.example.com; # names of your site
      # Change the values above to the appropriate values

      location ^~ /assets/ {
          root /path/to/pootle/;
      }

      location / {
          fastcgi_pass 127.0.0.1:8000;
          fastcgi_param QUERY_STRING $query_string;
          fastcgi_param REQUEST_METHOD $request_method;
          fastcgi_param CONTENT_TYPE $content_type;
          fastcgi_param CONTENT_LENGTH $content_length;
          fastcgi_param REQUEST_URI $request_uri;
          fastcgi_param DOCUMENT_URI $document_uri;
          fastcgi_param DOCUMENT_ROOT $document_root;
          fastcgi_param SERVER_PROTOCOL $server_protocol;
          fastcgi_param REMOTE_ADDR $remote_addr;
          fastcgi_param REMOTE_PORT $remote_port;
          fastcgi_param SERVER_ADDR $server_addr;
          fastcgi_param SERVER_PORT $server_port;
          fastcgi_param SERVER_NAME $server_name;
          fastcgi_pass_header Authorization;
          fastcgi_intercept_errors off;
          fastcgi_read_timeout 600;
      }
    }

.. note::

  The ``fastcgi_read_timeout`` line is only relevant if you're getting Gateway
  Timeout errors and you find them annoying. It defines how long (in seconds,
  default is 60) Nginx will wait for response from Pootle before giving up.
  Your optimal value will vary depending on the size of your translation
  project(s) and capabilities of the server.

.. note::

  Not all of these lines may be required. Feel free to remove those you find
  useless from this instruction.
