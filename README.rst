HAProxy plugin for Certbot
==========================

.. contents:: Table of Contents

About
-----

This is a certbot plugin for using certbot in combination with a HAProxy setup.
Its advantage over using the standalone certbot is that it automatically places
certificates in the correct directory and restarts HAProxy afterwards. It should
also enable you to very easily do automatic certificate renewal.

Furthermore, you can configure HAProxy to handle Boulder's authentication using
the HAProxy authenticator of this plugin.

It was created for use with `Greenhost's`_ shared hosting environment and can be
useful to you in the following cases:

- If you use HAProxy and have several domains for which you want to enable Let's
  Encrypt certificates.
- If you yourself have a shared hosting platform that uses HAProxy to redirect
  to your client's websites.
- Actually any case in which you want to automatically restart HAProxy after you
  request a new certificate.

.. _Greenhost's: https://greenhost.net

This plugin does not configure HAProxy for you, because HAProxy configurations
can vary a great deal. Please read the installation instructions on how to
configure HAProxy for use with the plugin. If you have a good idea on how we can
implement automatic HAProxy configuration, you are welcome to create a merge
request or an issue.

Installing: Requirements
------------------------

Currently this plugin has been tested on Debian Jessie, but it will most likely
work on Ubuntu 14.04+ too. If you are running Debian Wheezy, you may need to
take additional steps during the installation. Thus, the requirements are:

- Debian Jessie (or higher) or Ubuntu Trusty (or higher).
- Python 2.7 (2.6 is supported by certbot and our goal is to be compatible but
  it has not been tested yet).
- HAProxy 1.6+ (we will configure SNI, which is not strictly required)
- Certbot 0.8+

Installing: Getting started
---------------------------

The installation below assumes you are running Debian Jessie but it should be
almost entirely the same process on Ubuntu.

First add the backports repo for Jessie to your apt sources.

.. note::

    This will not work for Ubuntu, you will need to use another source,
    check which version comes with your version of Ubuntu, if it is a version
    below 0.8, you need to find a back port PPA or download certbot from source.

.. code:: bash

    echo "deb http://ftp.debian.org/debian jessie-backports main" >> \
        /etc/apt/sources.list.d/jessie-backports.list

Now update, upgrade and install some requirements:

- **Some utilities:** ``sudo`` ``tcpdump`` ``ufw`` ``git`` ``curl`` ``wget``
- **OpenSSL and CA certificates:** ``openssl`` ``ca-certificates``
- **Build dependencies:** ``build-essential`` ``libffi-dev`` ``libssl-dev`` ``python-dev``
- **Python and related:** ``python`` ``python-setuptools``
- **HAProxy:** ``haproxy``
- **Python dependency managing:** ``pip``

.. code:: bash

    apt-get update
    apt-get upgrade -y
    apt-get install -y \
        sudo tcpdump ufw git curl wget \
        openssl ca-certificates \
        build-essential libffi-dev libssl-dev python-dev \
        python python-setuptools \
        haproxy

    easy_install pip
    pip install --upgrade setuptools

We also installed a simple firewall above, but it is not yet configured, let's
do that now:

.. code:: bash

    ufw allow ssh
    ufw allow http
    ufw allow https
    ufw default deny incoming
    ufw --force enable

.. warning::

    You probably want a little more protection for a production proxy
    than just this simple firewall, but it's out of the scope of this readme.

Now that we have all dependencies, it's time to start a process that may take
quite some time to complete. HAProxy comes with a DH parameters file that is
considered weak. We need to generate a new dhparams.pem file with a prime of at
least ``2048`` bit length, you can also opt for ``3072`` or ``4096``. This can
take hours on lower specification hardware, but will still take minutes on
faster hardware, especially with ``4096`` bit primes. Run this is in a separate
ssh session or use ``screen`` of ``tmux`` to allow this to run in the
background.

.. code:: bash

    openssl dhparam -out /opt/certbot/dhparams.pem 2048

Now set a hostname.

.. code:: bash

    echo "[INSERT YOUR HOSTNAME HERE]" > /etc/hostname
    hostname -F /etc/hostname

If you want to run Certbot in an unprivileged mode, keep reading, otherwise,
skip to the installation of Certbot.

Certbot normally requires access to the ``/etc/`` directory, which is owned by
root and therefore, Certbot needs to run as root. However, we don't like it
when processes run as root, most especially when they are opening ports on a
public network interface..

In order to let Certbot run as an unprivileged user, we will:

- Create a ``certbot`` user with a home directory on the system so the
  automatic renewal of certificates can be run by this user.
- Tell Certbot that the working directories are located in ``certbot``'s home
  directory.
- Optionally: add your own user account to the Certbot user's group so you can
  run Certbot manually.
- Allow HAProxy to access the certificates that are generated by Certbot.
- Allow the certbot user to restart the HAProxy server.

Lastly, to do automatic renewal of certificates, we will create a systemd timer
and a service to start at every boot and every 12 hours, at a random time off
the day, in order to not collectively DDOS Let's Encrypts service.

.. code:: bash

    useradd -s /bin/bash -m -d /opt/certbot certbot
    usermod -a -G certbot haproxy  # Allow HAProxy access to the certbot certs
    mkdir -p /opt/certbot/logs
    mkdir -p /opt/certbot/config
    mkdir -p /opt/certbot/.config/letsencrypt

If you need to use Certbot from your user account, or if you have a daemon
running on your proxy server, that configures domains on your proxy, e.g.: in a
web hosting environment - you can add those users to the ``certbot`` group.

.. code:: bash

    usermod -a -G certbot [ADD YOUR USER HERE]

You will also need to tell your user what the working directory of your Certbot
setup is (``/opt/certbot/``). Certbot allows you to create a configuration file
with default settings in the users' home dir:
``opt/certbot/.config/letsencrypt/cli.ini``.

Besides the working directory.

.. code:: bash

    mkdir -p /opt/certbot/.config/letsencrypt
    cat <<EOF > /opt/certbot/.config/letsencrypt/cli.ini
    work-dir=/opt/certbot/
    logs-dir=/opt/certbot/logs/
    config-dir=/opt/certbot/config
    EOF

Next time you run Certbot, it will use our new working directory.

Now to allow the certbot user to restart HAProxy, put the following in the
sudoers file:

.. code:: bash

    cat <<EOF >> /etc/sudoers
    %certbot ALL=NOPASSWD: /bin/systemctl restart haproxy
    EOF

Now we haven't done one very essential thing yet, install ``certbot-haproxy``.
Since our plugin is in an alpha stage, we did not package it yet. You will need
to get it from our Gitlab server.

.. code:: bash

    git clone https://code.greenhost.net/open/certbot-haproxy.git
    cd ./certbot-haproxy/
    sudo pip install ./


Let's Encrypt's CA server will try to contact your proxy on port 80, which is
most likely in use for your and/or your customers' websites. So we have
configured our plugin to open port ``8000`` to verify control over the domain
instead. Therefore we need to forward verification requests on port 80 to port
8000 internally.

The sample below contains all that is required for a working load-balancing
HAProxy setup that also forwards these verification requests. But it is
probably not "copy-paste compatible" with your setup. So you need to piece
together a configuration that works for you.

.. code::

    cat <<EOF > /etc/haproxy/haproxy.cfg
    global
        log /dev/log local0
        log /dev/log local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default ciphers to use on SSL-enabled listening sockets.
        # Cipher suites chosen by following logic:
        #  - Bits of security 128>256 (weighing performance vs added security)
        #  - Key exchange: EECDH>DHE (faster first)
        #  - Mode: GCM>CBC (streaming cipher over block cipher)
        #  - Ephemeral: All use ephemeral key exchanges
        #  - Explicitly disable weak ciphers and SSLv3
        ssl-default-bind-ciphers AES128+AESGCM+EECDH+SHA256:AES128+EECDH:AES128+AESGCM+DHE:AES128+EDH:AES256+AESGCM+EECDH:AES256+EECDH:AES256+AESGCM+EDH:AES256+EDH:-SHA:AES128+AESGCM+EECDH+SHA256:AES128+EECDH:AES128+AESGCM+DHE:AES128+EDH:AES256+AESGCM+EECDH:AES256+EECDH:AES256+AESGCM+EDH:AES256+EDH:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!3DES:!DSS
        #ssl-default-bind-options no-sslv3 no-tls-tickets force-tlsv12
        ssl-default-bind-options no-sslv3 no-tls-tickets
        ssl-dh-param-file /opt/certbot/dhparams.pem

    defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

    frontend http-in
        # Listen on port 80
        bind \*:80
        # Listen on port 443
        # Uncomment after running certbot for the first time, a certificate
        # needs to be installed *before* HAProxy will be able to start when this
        # directive is not commented.
        #
        bind \*:443 ssl crt /opt/certbot/haproxy_fullchains/__fallback.pem crt /opt/certbot/haproxy_fullchains

        # Forward Certbot verification requests to the certbot-haproxy plugin
        acl is_certbot path_beg -i /.well-known/acme-challenge
        rspadd Strict-Transport-Security:\ max-age=31536000;\ includeSubDomains;\ preload
        rspadd X-Frame-Options:\ DENY
        use_backend certbot if is_certbot
        # The default backend is a cluster of 4 Apache servers that you need to
        # host.
        default_backend nodes

    backend certbot
        log global
        mode http
        server certbot 127.0.0.1:8000

        # You can also configure separate domains to force a redirect from port 80
        # to 443 like this:
        # redirect scheme https if !{ ssl_fc } and [PUT YOUR DOMAIN NAME HERE]

    backend nodes
        log global
        balance roundrobin
        option forwardfor
        option http-server-close
        option httpclose
        http-request set-header X-Forwarded-Port %[dst_port]
        http-request add-header X-Forwarded-Proto https if { ssl_fc }
        option httpchk HEAD / HTTP/1.1\r\nHost:localhost
        server node1 127.0.0.1:8080 check
        server node2 127.0.0.1:8080 check
        server node3 127.0.0.1:8080 check
        server node4 127.0.0.1:8080 check
        # If redirection from port 80 to 443 is to be forced, uncomment the next
        # line. Keep in mind that the bind \*:443 line should be uncommented and a
        # certificate should be present for all domains
        redirect scheme https if !{ ssl_fc }

    EOF

    systemctl restart haproxy

Now you can try to run Certbot with the plugin as the Authenticator and
Installer, if you already have websites configured in your HAProxy setup, you
may try to install a certificate now.

.. code:: bash

    certbot run --authenticator certbot-haproxy:haproxy-authenticator \
        --installer certbot-haproxy:haproxy-installer

If you want your ``certbot`` to always use our Installer and Authenticator, you
can add this to your configuration file:

.. code:: bash

    cat <<EOF >> $HOME/.config/letsencrypt/cli.ini
    authenticator=certbot-haproxy:haproxy-authenticator
    installer=certbot-haproxy:haproxy-installer
    EOF

If you need to run in unattended mode, there are a bunch of arguments you need
to set in order for Certbot to generate a certificate for you.

- ``--domain [DOMAIN NAME]`` The domain name you want SSL to be enabled for.
- ``--agree-tos`` Tell Certbot you agree with its `TOS`_
- ``--email [EMAIL ADDRESS]`` An e-mail address where issues with certificates
  can be sent to, as well as changes in the `TOS`_. Or you could supply
  ``--register-unsafely-without-email`` but this is not recommended.

.. _TOS: https://letsencrypt.org/documents/LE-SA-v1.1.1-August-1-2016.pdf

After you run certbot successfully once, there will be 2 certificate files in
the certificate directory. This is a pre-requisite for HAProxy to start with
the ``bind *:443 [..]`` directive in the configuration.

You can auto renew certificates by using the systemd service and timer below.
They are set to run every 12 hours because certificates that *will not* expire
soon will not be replaced but certificates that *will* expire soon, will be
replaced in a timely manner. The timer also starts the renewal process 2
minutes after the server boots, this is done so renewal starts immediately
after the server has been offline for a long time.

.. code:: bash

    cat <<EOF > /etc/systemd/system/letsencrypt.timer
    [Unit]
    Description=Run Let's Encrypt every 12 hours

    [Timer]
    # Time to wait after booting before we run first time
    OnBootSec=2min
    # Time between running each consecutive time
    OnUnitActiveSec=12h
    Unit=letsencrypt.service

    [Install]
    WantedBy=timers.target
    EOF

    cat <<EOF > /etc/systemd/system/letsencrypt.service
    [Unit]
    Description=Renew Let's Encrypt Certificates

    [Service]
    Type=simple
    User=certbot
    ExecStart=/usr/bin/certbot renew -q
    EOF

    # Enable the timer and start it, this is not necessary for the service,
    # since the timer starts it.
    systemctl enable letsencrypt.timer
    systemctl start letsencrypt.timer


Development: Getting started
-----------------------------

In order to run tests against the Let's Encrypt API we will run a Boulder
server, which is the exact same server Let's Encrypt is running. The server is
started in Virtual Box using Vagrant. To prevent the installation of any
components and dependencies from cluttering up your computer there is also a
client Virtual Box instance. Both of these machines can be setup and started by
running the ``dev_start.sh`` script. This sets up a local boulder server and the
letsencrypt client, so don't worry if it takes more than an hour.

Vagrant machines
================
The ``dev_start.sh`` script boots two virtual machines. The first is named
'boulder' and runs a development instance of the boulder server. The second is
'lehaproxy' and runs the client. To test if the machines are setup correctly,
you can SSH into the 'lehaproxy' machine, by running ``vagrant ssh
lehaproxy``. Next, go to the /lehaproxy directory and run
``./tests/boulder-integration.sh``. This runs a modified version of certbot's
boulder-integration test, which tests the HAProxy plugin. If the test succeeds,
your development environment is setup correctly.

Development: Running locally without sudo
-----------------------------------------

You can't run certbot without root privileges because it needs to access
``/etc/letsencrypt``, however you can tell it not to use ``/etc/`` and use some
other path in your home directory.

.. code:: bash

    mkdir ~/projects/certbot-haproxy/working
    mkdir ~/projects/certbot-haproxy/working/config
    mkdir ~/projects/certbot-haproxy/working/logs
    cat <<EOF >> ~/.config/letsencrypt/cli.ini
    work-dir=~/projects/certbot-haproxy/working/
    logs-dir=~/projects/certbot-haproxy/working/logs/
    config-dir=~/projects/certbot-haproxy/working/config
    EOF

Now you can run Certbot without root privileges.

Further time savers during development..
----------------------------------------
The following options can be saved in the ``cli.ini`` file for the following
reasons.

- ``agree-tos``: During each request for a certificate you need to agree to the
  terms of service of Let's Encrypt, automatically accept them every time.
- ``no-self-upgrade``: Tell LE to not upgrade itself. Could be very annoying
  when stuff starts to suddenly break, that worked just fine before.
- ``register-unsafely-without-email``: Tell LE that you don't want to be
  notified by e-mail when certificates are about to expire or when the TOS
  changes, if you don't you will need to enter a valid e-mail address for
  every test run.
- ``text``: Disable the curses UI, and use the plain CLI version instead.
- ``domain example.org``: Enter a default domain name to request a certificate
  for, so you don't have to specify it every time.
- ``configurator certbot-haproxy:haproxy``: Test with the HAProxy plugin every
  time.

.. code:: bash

    cat <<EOF >> ~/.config/letsencrypt/cli.ini
    agree-tos=True
    no-self-upgrade=True
    register-unsafely-without-email=True
    text=True
    domain=example.org
    authenticator=certbot-haproxy:haproxy-authenticator
    installer=certbot-haproxy:haproxy-installer
    EOF

Setuptools version conflict
---------------------------

Most likely the ``python-setuptools`` version in your os's repositories is
quite outdated. You will need to install a newer version, to do this you can
run:

.. code:: bash

    pip install --upgrade setuptools

Since pip is part of ``python-setuptools``, you need to have it installed before
you can update.

Making a `.deb` debian package
------------------------------

Requirements:

- python stdeb: pip install --upgrade stdeb
- dh clean: apt-get install dh-make

Run the following commands in your vagrant machine:

.. code:: bash

    apt-file update
    python setup.py sdist
    # py2dsc has a problem with vbox mounted folders
    mv dist/certbot-haproxy-<version>.tar.gz ~
    cd ~
    py2dsc certbot-haproxy-<version>.tar.gz
    cd deb_dist/certbot-haproxy-<version>
    # NOTE: Not signed, no signed changes (with -uc and -us)
    # NOTE: Add the package to the ghtools repo
    dpkg-buildpackage -rfakeroot -uc -us
