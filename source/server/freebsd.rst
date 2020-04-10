.. _install-freebsd:

FreeBSD Server
==============

It is very easy to install oTree on FreeBSD. This allows you to leverage many of the
unique features of FreeBSD, such as jails. **This installation guide is for expert system
administrators only.** If you don't consider yourself a sysadmin, please refer to the
other guides: :ref:`Windows <install-windows>`, :ref:`MacOS <install-macos>`,
:ref:`Linux <install-linux>`.

Note that many warnings of the :ref:`Ubuntu installation guide <server-ubuntu>` apply.
Many sections about actually running the server have been omitted, such as using a
reverse proxy. Still, they might be important; they are merely not contained here.

We assume that you have already installed ``sudo`` and enabled members of the group
``wheel`` to execute any command. If you want to allow each oTree user on the system
to install required packages, you should add them to ``wheel``.

Install required packages and oTree itself
------------------------------------------

Become superuser (you must be in the ``wheel`` group to do so)::

    su -

Run::

    pkg install py37-pip postgresql12-client postgresql12-server redis python37 py37-sqlite3 git
    
    pip install otree psycopg2

Database (PostgreSQL)
---------------------

oTree's default database is SQLite, which is fine for local development,
but insufficient for production.
We recommend you use PostgreSQL.

As superuser, you first have to initialize and start PostgreSQL::

    /usr/local/etc/rc.d/postgresql initdb
    service postgresql enable
    service postgresql start

Change users to the ``postgres`` user, so that you can execute some commands::

    su - postgres

Then start the Postgres shell::

    psql

Once you're in the shell, create a database and user::

    CREATE DATABASE django_db;
    alter user postgres password 'password';

Exit the SQL prompt::

    \q

Return to your regular command prompt (root)::

    exit

For each user that will use oTree, please
add this line to the end of the user's .bash_profile::

    export DATABASE_URL=postgres://postgres:password@localhost/django_db

Once ``DATABASE_URL`` is defined, oTree will use it instead of the default SQLite.

When you run ``otree resetdb`` later,
if you get an error that says "password authentication failed for user",
find your ``hba_auth.conf`` file, and on the lines for ``IPv4`` and ``IPv6``,
change the ``METHOD`` from ``md5`` (or whatever it currently is) to ``trust``.

Install Redis
-------------

If you installed Redis through ``pkg`` as instructed earlier, you should enable
and start Redis like so::

    service redis enable
    service redis start

Then,
Redis should be running on port 6379. You can test with ``redis-cli ping``,
which should output ``PONG``.

Create a virtualenv
-------------------

It's a best practice to use a virtualenv. Please return to the normal user
that wants to use oTree and run::

    python3 -m venv venv_otree

To activate this venv every time you start your shell, put this at the end of the user's ``.bash_profile``::

    source ~/venv_otree/bin/activate

Once your virtualenv is active, you will see ``(venv_otree)`` at the beginning
of your prompt.

Push your code to the server
----------------------------

You can get your code on the server using SCP, SFTP, rsync, git, etc.

For this tutorial, we will assume you are storing your files under
``/home/my_username/oTree``.

Reset the database on the server
--------------------------------

On the server, ``cd`` to the folder containing your oTree project.
Install the requirements and reset the database::

    sudo pip3 install -r requirements.txt
    otree resetdb

Running the server
------------------

If you are just testing your app locally, you can use the usual ``zipserver`` or ``devserver``
command.

However, when you want to use oTree in production, you need to run the
production server, which can handle more traffic.

Testing the production server
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

From your project folder, run::

    otree runprodserver 8000

Then navigate in your browser to your server's
IP/hostname followed by ``:8000``.

If you're not using a reverse proxy like Nginx or Apache,
you probably want to run oTree directly on port 80.
This requires superuser permission, so let's use sudo,
but add some extra args to preserve environment variables like ``PATH``,
``DATABASE_URL``, etc::

    sudo -E env "PATH=$PATH" otree runprodserver 80

Try again to open your browser;
this time, you don't need to append :80 to the URL, because that is the default HTTP port.

Notes:

-   unlike ``devserver``, ``runprodserver`` does not restart automatically
    when your files are changed.
-   ``runprodserver`` automatically runs Django's ``collectstatic``
    to collect your files under ``_static_root/``.
    If you have already run ``collectstatic``, you can skip it with
    ``--no-collectstatic``.

Set remaining environment variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add these in the same place where you set ``DATABASE_URL``::

    export OTREE_ADMIN_PASSWORD=my_password
    #export OTREE_PRODUCTION=1 # uncomment this line to enable production mode
    export OTREE_AUTH_LEVEL=DEMO
