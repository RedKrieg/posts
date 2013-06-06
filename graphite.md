Installing Graphite with Ceres and Megacarbon
=============================================

Graphite has changed much in the last year, though you wouldn't know it from reading the official site.  Much work has gone in to the new metric writer, [Ceres](https://github.com/graphite-project/ceres), to improve on the already fantastic Whisper backend which has been the default in the past.  Ceres allocates space as metrics are recorded, so your storage needs will scale more directly with use than with Whisper.  Additionally, the various carbon daemons (carbon-cache, carbon-relay, etc.) have all been merged into a single daemon called "Megacarbon".  This new daemon supports all the same functionality but streamlines the configuration process.  These instructions are for Ubuntu Server 13.04, but you can probably adapt them to your distro of choice if you have sufficient experience.

To start with, you'll need to get some tools and libraries to help you configure everything.  These will need to be installed system-wide:

    sudo aptitude install git python-pip python-dev libcairo2 libffi-dev memcached uwsgi nginx uwsgi-plugin-python uwsgi-plugin-carbon
    sudo pip install -U pip # Get the latest pip
    sudo pip install -U virtualenvwrapper # Get the latest virtualenvwrapper

Now that you've got the requirements installed, you'll need to create a user for graphite:

    sudo adduser graphite

You'll be prompted for details about the user, enter anything you wish.  I recommend a 64 or more character password, and that you do not bother to record it.  With sudo access to the box, you will not need it.  The next step is creating the graphite storage location.  I prefer /opt/graphite for this:

    sudo mkdir /opt/graphite
    sudo chown -R graphite.graphite /opt/graphite
    sudo chmod 751 /opt/graphite # More on this later

Now switch to your graphite user and configure virtualenv wrapper:

    sudo su - graphite
    source /usr/local/bin/virtualenvwrapper.sh
    echo "source /usr/local/bin/virtualenvwrapper.sh" >> .bashrc

Once virtualenvwrapper is configured, you can create the new virtualenv and start installing graphite:

    mkvirtualenv -a /opt/graphite graphite # This should change your directory to /opt/graphite

We need a directory for our sources:

    mkdir src
    cd src/

From the src folder, we can start installing graphite components:

    export GRAPHITE_ROOT=/opt/graphite
    git clone git://github.com/graphite-project/carbon.git -b megacarbon
    cd carbon/
    pip install -r requirements.txt
    python setup.py install
    cd $GRAPHITE_ROOT/src/
    git clone git://github.com/graphite-project/ceres.git
    pip install -r requirements.txt
    python setup.py install

Next, configure Ceres' datastore:

    ceres-tree-create /opt/graphite/storage/ceres

Now we need to edit config files.  Start by copying the example configs:

    cd $GRAPHITE_ROOT/conf/carbon-daemons/
    cp -r example/ writer
    cd writer/

The three files that you should confirm are configured as you desire are **daemon.conf**, **writer.conf**, and **db.conf**.  It's a good idea to read the others as well.  The parts you *must* change are:

    in daemon.conf:
    USER = graphite

    in db.conf:
    DATABASE = ceres
    LOCAL_DATA_DIR = /opt/graphite/storage/ceres/

The other options can remain the same should you so desire.  Now we install the web app:

    cd $GRAPHITE_ROOT/src/
    git clone git://github.com/graphite-project/graphite-web.git
    cd graphite-web/

Unfortunately, I wasn't able to get pycairo to install properly inside a virtualenv, so I used a drop-in replacement library called [cairocffi](http://pythonhosted.org/cairocffi/) which works flawlessly as far as I can tell.  Remove pycairo from the requirements, and install the remainder along with cairocffi:

    sed -i '/cairo/d' requirements.txt
    pip install -r requirements.txt
    pip install cairocffi

Next, we have a small hack to ensure cairocffi gets used instead of pycairo:

    echo 'from cairocffi import *' > /home/graphite/.virtualenvs/graphite/lib/python2.7/site-packages/cairo.py

Finally, install graphite-web:

    python setup.py install

Now you need an entry point for your application:

    cd $GRAPHITE_ROOT/webapp/

Now create **wsgi.py** in this folder and populate it with the following text:

    import os, sys
    os.environ['DJANGO_SETTINGS_MODULE'] = 'graphite.settings'
    
    import django.core.handlers.wsgi
    
    app = django.core.handlers.wsgi.WSGIHandler()

This file will be the module imported by uwsgi to run your app.  You should take the opportunity now to configure your database.  You can create a superuser now as well, who will be the user you log into the application as:

    cd cd $GRAPHITE_ROOT/webapp/graphite/
    python manage.py syncdb

We're nearly there.  Now we get everything starting on boot and configure nginx/uwsgi:

    exit # So we're not running as the graphite user

Edit **/etc/init/carbon-writer.conf** and place the following inside:

    description "carbon writer"
     
    start on runlevel [2345] stop on runlevel [016]
     
    expect daemon
     
    respawn limit 5 10
     
    env GRAPHITE_DIR=/opt/graphite
    exec start-stop-daemon --oknodo --pidfile /var/run/carbon-writer.pid --startas $GRAPHITE_DIR/bin/carbon-daemon.py --start writer start

Now start the carbon daemon and ensure it's listening:

    sudo start carbon-writer
    sudo netstat -plan | grep :2003

You should see a process listening on port 2003.  Once that's up, you can configure the webserver.  Edit **/etc/nginx/sites-available/graphite** and enter the following:

    server {
      listen 80;
      keepalive_timeout 60;
      server_name graphite.example.com;
      charset utf-8;
      location / {
        include uwsgi_params;
        uwsgi_param UWSGI_CHDIR /opt/graphite/webapp/;
        uwsgi_param UWSGI_PYHOME /home/graphite/.virtualenvs/graphite/;
        uwsgi_param UWSGI_MODULE wsgi;
        uwsgi_param UWSGI_CALLABLE app;
        uwsgi_pass 127.0.0.1:3031;
      }
      location /content/ {
        alias /opt/graphite/webapp/content/;
        autoindex off;
      }
    }

Naturally you should substitute *example.com* for your actual domain.  Now we ensure nginx can read the content directory (this is why we set /opt/graphite to permission 751 earlier):

    sudo chown -R graphite.www-data /opt/graphite/webapp/content
    sudo chmod 751 /opt/graphite/webapp /opt/graphite/webapp/content

Now we create a configuration file for uwsgi.  Edit **/etc/uwsgi/apps-available/graphite.ini** and enter the following:

    [uwsgi]
    plugins = python
    gid = graphite
    uid = graphite
    vhost = true
    logdate
    socket = 127.0.0.1:3031
    master = true
    processes = 4
    harakiri = 20
    limit-as = 256
    wsgi-file = /opt/graphite/webapp/wsgi.py
    virtualenv = /home/graphite/.virtualenvs/graphite
    chdir = /opt/graphite
    memory-report
    no-orphans
    carbon = 127.0.0.1:2003

Note the *carbon* directive, which tells uwsgi to graph some metrics about accesses, memory, etc.  This will be our test data to ensure everything works.

By default in Ubuntu, uwsgi will run as the www-data user and cannot change privileges to the graphite user.  To work around this, we can modify the default configuration at **/usr/share/uwsgi/conf/default.ini** and remove the uid and gid lines.  This will allow the graphite.ini file we just created to drop privileges to the graphite user.

We can now enable the application and load the configurations:

    sudo ln -s /etc/nginx/sites-{available,enabled}/graphite
    sudo ln -s /etc/uwsgi/apps-{available,enabled}/graphite.ini
    sudo service nginx restart
    sudo service uwsgi restart

If everything went well, you should now be able to reach your site at the url you configured.  When you expand the *Graphite* group you should see a *uwsgi* group available that contains metrics about the running instance of uwsgi (graphite itself).
