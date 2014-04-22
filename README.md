metpetdb-py
===========

Python- Django implementation of the Metpetdb System (metpetdb.rpi.edu)

Currently, we have the webservices for faceted search.

Plans to expand to have webservices for read/edit/delete samples.

Currently the stable system has been implemented in java.

Python and virtualenv setup
---------------------------

If there is no Python 2.7.x on your system, install it

    sudo apt-get install python2.7

####Virtual environment setup

    mkdir ~/.virtualenvs
    cd ~/.virtualenvs

Get the latest virtualenv stable tarball and extract it. The directory name into which the tarball gets extracted might be different than the one mentioned here.

    curl -Lo virtualenv-tmp.tar.gz 'https://github.com/pypa/virtualenv/tarball/master'
    tar -xzvf virtualenv-tmp.tar.gz

Install a base virtual environment: env0

    python pypa-virtualenv-0b71587/virtualenv.py env0

Install the virtualenv package into our new base environment

    env0/bin/pip install virtualenv-tmp.tar.gz
    rm virtualenv-tmp.tar.gz

Add the following lines to ~/.bashrc
```bash
function mkvirtualenv {
~/.virtualenvs/env0/bin/virtualenv ~/.virtualenvs/$1
}

function workon {
source ~/.virtualenvs/$1/bin/activate
}
```

    source ~/.bashrc

Usage

    mkvirtualenv test_env
    workon test_env


Postgres installation
----------------------

Install postgresql (ver 9.1.x)

    sudo apt-get install postgresql

Update 'pg_hba.conf' file to authenticate connections

```
cd /etc/postgresql/9.1/main
sudo vi pg_hba.conf
```

```
# Database administrative login by Unix domain socket
local   all             postgres                                peer
local   all     postgres                md5

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             128.213.23.26/32        md5
host    all     all     192.168.1.32/32     md5
host    all     all     127.0.0.1       md5

# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5
```

Edit the postgresql.conf file (in the same folder as pg_hba)

```
#change listen_address to allow remote connections
listen_addresses = '*'          # what IP address(es) to listen on;
```

Restart the Postgres server for the above changes to take effect

    sudo service postgresql restart

Set the `postgres` username/password. Be careful with password since it cannot be retrived, only reset.

    sudo -u postgres psql template1
    \password postgres

Create a new login role `metpetdb` and set the password

```sql
CREATE ROLE metpetdb INHERIT LOGIN CREATEDB PASSWORD 'metpetdb';
```

Exit the psql front-end by entering `\q`

If it isn't already there, register the procedural language `plpgsql` to implement stored procedures. Switch to the `postgres` user first. Set a new password for postgres if you don't know the current password.

    sudo passwd postgres
    su postgres
    createlang plpgsql


PostGIS installation
---------------------

Before we begin, you uninstall any existing PostGIS packages:

    sudo dpkg --purge postgis postgresql-9.1-postgis

####Build GEOS 3.4.x

    wget http://download.osgeo.org/postgis/source/postgis-2.1.2.tar.gz
    tar xfz postgis-2.1.2.tar.gz
    cd postgis-2.1.2

####Build PostGIS
    wget http://download.osgeo.org/postgis/source/postgis-2.1.2.tar.gz
    tar xfz postgis-2.1.2.tar.gz
    cd postgis-2.1.2

    ./configure
    make
    sudo make install
    sudo ldconfig
    sudo make comments-install

####Database setup

Create a PostGIS template

    su postgres
    createdb -E UTF8 template_postgis
    createlang -d template_postgis plpgsql

    psql -d postgres -c "UPDATE pg_database SET datistemplate='true' WHERE datname='template_postgis'"
    psql -d template_postgis -f /usr/share/postgresql/9.1/contrib/postgis-2.1/postgis.sql
    psql -d template_postgis -f /usr/share/postgresql/9.1/contrib/postgis-2.1/spatial_ref_sys.sql

    psql -d template_postgis -c "GRANT ALL ON geometry_columns TO PUBLIC;"
    psql -d template_postgis -c "GRANT ALL ON geography_columns TO PUBLIC;"
    psql -d template_postgis -c "GRANT ALL ON spatial_ref_sys TO PUBLIC;"

Create a database named metpetdb from the postgis template: template_postgis. You can choose any database name you want, but remember to make corresponding changes in `metpetdb/settings.py` once you check out the codebase.

    createdb metpetdb -T template_postgis

Change ownership -- you can do this via pgAdmin as well

```sql
ALTER DATABASE metpetdb OWNER TO metpetdb;
ALTER view geometry_columns OWNER TO metpetdb;
ALTER view geography_columns OWNER TO metpetdb;
ALTER table spatial_ref_sys OWNER TO metpetdb;
```

Sample command to import the data dump into the database (this data dump isn't publicly available)

    cat test_sed_jan23_2013.dmp  | psql -U metpetdb -dmetpetdb


Codebase setup
-----------------------

    git clone git@github.com:metpetdb/metpetdb-py.git code
    cd code
    mkvirtualenv mpdb
    pip install -r requirements.txt

####Database migration scripts

    cd database/setup_sql/
    cat all.sql | psql -U metpetdb -dmetpetdb

####Transition scipt

The transition script does two things:
* Transition user data from the old database to a Django-compatible format.
* Create appropriate database records for samples, subsamples, chemical analyses, images, and grids so that they can be accessed through the application's access control system.

First, copy the following function into `~/.bashrc`:
```bash
function setdsm() {
    # add the current directory and the parent directory to PYTHONPATH
    # sets DJANGO_SETTINGS_MODULE
    export PYTHONPATH=$PYTHONPATH:$PWD/..
    export PYTHONPATH=$PYTHONPATH:$PWD
    if [ -z "$1" ]; then
        x=${PWD/\/[^\/]*\/}
        export DJANGO_SETTINGS_MODULE=$x.settings
    else
        export DJANGO_SETTINGS_MODULE=$1
    fi

    echo "DJANGO_SETTINGS_MODULE set to $DJANGO_SETTINGS_MODULE"
}
```

And then run the script

    source ~/.bashrc
    cd metpetdb/
    setdsm
    cd ../tastyapi
    python transition.py

If you are developing locally, you can now start the server and start hacking:

    python manage.py runserver

Apache web server installation
------------------------------

####Install Apache
    sudo apt-get install apache2
    sudo apt-get install apache2-threaded-dev python2.7-dev

####Install mod_wsgi
    wget http://modwsgi.googlecode.com/files/mod_wsgi-3.4.tar.gz
    tar xvfz mod_wsgi-3.4.tar.gz
    cd mod_wsgi-3.4
    ./configure
    make
    sudo make install
    echo "LoadModule wsgi_module /usr/lib/apache2/modules/mod_wsgi.so" | sudo tee /etc/apache2/mods-available/wsgi.load
    sudo a2enmod wsgi
    sudo a2dissite default
    sudo service apache2 restart

Setup Apache to serve the application
-------------------------------------

Add read and execute for "others" on the codebase and the Python virtual environment

    chmod -R o+rx ~/code
    chmod -R o+rx ~/.virtualenvs/mpdb

    sudo vi /etc/apache2/httpd.conf

Paste the following lines in `http.conf`

    WSGIScriptAlias / /home/metpetdb_py/code/apache/wsgi.py
    WSGIPythonPath /home/metpetdb_py/code:/home/metpetdb_py/.virtualenvs/mpdb/lib/python2.7/site-packages

    <Directory /home/metpetdb_py/code/apache>
    <Files wsgi.py>
    Order deny,allow
    Allow from all
    </Files>
    </Directory>

Paste the following lines in ~/code/apache/wsgi.py

    import os
    import sys

    path = '/home/metpetdb_py/code'
    if path not in sys.path:
        sys.path.append(path)
    sys.path.append('/home/metpetdb_py')
    sys.path.append('/home/metpetdb_py/.virtualenvs/mpdb/lib/python2.7/site-packages')
    os.environ['DJANGO_SETTINGS_MODULE'] = 'metpetdb.settings'

    activate_this = '/home/metpetdb_py/.virtualenvs/mpdb/bin/activate_this.py'
    execfile(activate_this, dict(__file__=activate_this))

    import django.core.handlers.wsgi
    application = django.core.handlers.wsgi.WSGIHandler()

```sudo service apache2 restart```

You can now start interacting with the API at `http://hostname/api/vi/`

***



Server setup instructions end here, what follows are legacy instructions and will eventually either be edited/deleted.

if interested check out the existing webservices for faceted search from Git
https://github.com/metpetdb/metpetdb-py
 documentation regarding this is here https://github.com/metpetdb/metpetdb-py/tree/master/docs

if you want to have them running, so an svn checkout or Git pull
 you may have to set up svn repository (refer: https://help.ubuntu.com/community/Subversion)
write models or generate with
python mysite/manage.py inspectdb > mysite/myapp/models.py

update models
- rearrange classes
- add related_names (if required)

check SQL and validate
python manage.py sql webservices
python manage.py validate

create views
set urls inside metpetdb/urls.py

Run the webservices on your development server with the following command:
python manage.py runserver

