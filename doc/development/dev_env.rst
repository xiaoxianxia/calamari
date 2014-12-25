

Setting up a development environment
====================================

These are *not* the instructions for installing Calamari, or for building Calamari packages.  These
are the instructions for setting up a development instance of Calamari that runs as an unprivileged
user from your development workstation.

Using Vagrant
-------------

.. note::

    We assume some familiarity with vagrant: check out the `vagrant tutorial <http://docs.vagrantup.com/v2/getting-started/>`_
    if you are new to vagrant.

1. Make sure you have installed a recent version of Vagrant -- the versions bundled with linux
   distributions tend to be old, it is better to get it from http://www.vagrantup.com/
2. If you are using a vagrant provider other than virtualbox, seek out a ``precise64`` box
   for your provider (you may find one at http://www.vagrantbox.es/)
3. ``cd vagrant/devmode``
4. ``vagrant up``

The result will be a virtual machine that you can connect to using ``vagrant ssh``, containing
a git clone of Calamari with a fully populated virtualenv.  Activate the virtualenv

There is a screencast of this process: https://www.youtube.com/watch?v=Nil70DgL2zg


By hand
-------

.. warning::

    Hey!  Are you sure you meant to be here?  This is a lengthy manual procedure that just
    replicates the environment you would get from using the vagrant configuration.


Installing dependencies
_______________________

If you haven't already, install some build dependencies (apt, yum, brew, whatever) which
will be needed by the pip packages:

::

    sudo apt-get install python-virtualenv git python-dev swig libzmq-dev g++ postgresql-9.1 postgresql-server-dev-9.1

If you're on ubuntu, there are a couple packages that are easier to install with apt
than with pip (and because of `m2crypto weirdness`_)

::

    sudo apt-get install python-cairo python-m2crypto

For RH systems:

::

    sudo yum  install python-virtualenv git python-devel swig zeromq-devel gcc-c++ postgresql-server postgresql-devel pycairo m2crypto


1. Create a virtualenv (if you are on ubuntu and using systemwide installs of
   cairo and m2crypto, then pass *--system-site-packages*)

::

     cd calamari
     virtualenv --system-site-packages calamari_venv
     source calamari_venv/bin/active 
     export VIRTUAL_ENV=`pwd`/calamari_venv

*You can use other virtualenv name for {calamari_venv}*
    
2. Install dependencies with ``pip install -r requirements/{debian,rh}/requirements.txt`` and ``pip install -r requirements/{debian,rh}/requirements.force.txt``.
3. Install graphite and carbon, which require some special command lines:

::

    pip install carbon --install-option="--prefix=$VIRTUAL_ENV" --install-option="--install-lib=$VIRTUAL_ENV/lib/python2.7/site-packages"
    pip install git+https://github.com/ceph/graphite-web.git@calamari --install-option="--prefix=$VIRTUAL_ENV" --install-option="--install-lib=$VIRTUAL_ENV/lib/python2.7/site-packages"


4. Grab the `GUI code <https://github.com/ceph/calamari-clients>`_, build it and
   place the build products in ``webapp/content`` so that when it's installed you
   have a ``webapp/content`` directory containing ``admin``, ``dashboard`` and ``login``.

.. _m2crypto weirdness: http://blog.rectalogic.com/2013/11/installing-m2crypto-in-python.html

*Aside: unless you like waiting around, set PIP_DOWNLOAD_CACHE in your environment*

Getting ready to run
____________________

Calamari server consists of multiple modules, link them into your virtualenv:

::

    pushd calamari-common ; python setup.py develop ; popd
    pushd rest-api ; python setup.py develop ; popd
    pushd cthulhu ; python setup.py develop ; popd
    pushd minion-sim ; python setup.py develop ; popd
    pushd calamari-web ; python setup.py develop ; popd

Graphite needs some folders created:

::

    mkdir -p ${VIRTUAL_ENV}/storage/log/webapp
    mkdir -p ${VIRTUAL_ENV}/storage


The development-mode config files have some absolute paths that need rewriting in
a fresh checkout, there's a script for this:

::

    ~/calamari$ dev/configure.py
    Calamari repo is at: /home/vagrant/calamari, user is vagrant
    Writing /home/vagrant/calamari/dev/etc/salt/master
    Writing /home/vagrant/calamari/dev/calamari.conf
    Complete.  Now run:
     1. `CALAMARI_CONFIG=dev/calamari.conf calamari-ctl initialize`
     2. supervisord -c dev/supervisord.conf -n


Set up postgres, the easiest way to do this is by using salt:

::

    ~/calamari$ sudo salt-call --local state.template salt/local/postgres.sls

*Before do this,make sure you postgressql is running sucessful*

If you want to create your database another way, examine the .sls file to see
the settings that are expected.  If you want to use another database like SQLite,
you can edit the paths in calamari.conf.

Create the cthulhu service's database:

::

    CALAMARI_CONFIG=dev/calamari.conf calamari-ctl initialize


Aside: the ``CALAMARI_CONFIG`` environment variable causes all the calamari services to
read their configuration from an alternative location.  In a package-installed system
the config is loaded from /etc/calamari/calamari.conf.  We use an overridden config file
in development instead of having any "if DEBUG" flags anywhere in the code.


Running the server
__________________

The server processes are run for you by ``supervisord``.  A healthy startup looks like this:

::

    calamari john$ supervisord -n -c dev/supervisord.conf
    2013-12-02 10:26:51,922 INFO RPC interface 'supervisor' initialized
    2013-12-02 10:26:51,922 CRIT Server 'inet_http_server' running without any HTTP authentication checking
    2013-12-02 10:26:51,923 INFO supervisord started with pid 31453
    2013-12-02 10:26:52,925 INFO spawned: 'salt-master' with pid 31456
    2013-12-02 10:26:52,927 INFO spawned: 'carbon-cache' with pid 31457
    2013-12-02 10:26:52,928 INFO spawned: 'calamari-frontend' with pid 31458
    2013-12-02 10:26:52,930 INFO spawned: 'cthulhu' with pid 31459
    2013-12-02 10:26:54,435 INFO success: salt-master entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2013-12-02 10:26:54,435 INFO success: carbon-cache entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2013-12-02 10:26:54,435 INFO success: calamari-frontend entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2013-12-02 10:26:54,435 INFO success: cthulhu entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)

Supervisor will print complaints if something is not starting up properly.  Check in the various \*.log files to
find out why something is broken, or run processes individually by hand to debug them (see the commands in supervisord.conf).

At this point you should have a server up and running at ``http://localhost:8000/`` and
be able to log in to the UI.

Connecting Ceph servers to Calamari
-----------------------------------

Simulated minions
_________________

Impersonate some Ceph servers with the minion simulator:

::

    minion-sim --count=3


Real minions
____________

If you have a real live Ceph cluster, install ``salt-minion`` on each of the
servers, and configure it to point to your development instance host


1.install salt-minion

You can get salt-minion(2014.1.10) here https://launchpad.net/~saltstack/+archive/ubuntu/salt-depends/+sourcepub/4335419/+listing-archive-extra

2.build and install diamond

3.config salt-minion

::

    echo "master: fqdn" > /etc/salt/minion.d/calamari.conf
    salt-minion restart

Allowing minions to join
________________________

Authorize the simulated salt minions to connect to the calamari server:

::

    salt-key -c dev/etc/salt -L
    salt-key -c dev/etc/salt -A

You should see some debug logging in cthulhu.log, and if you visit /api/v2/cluster in your browser
a Ceph cluster should appear.
