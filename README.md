<img src="https://cloud.githubusercontent.com/assets/22897558/22356169/8f6e0b16-e3ec-11e6-8695-2273424c2b06.png" width="200" />

# Overview 

PcapDB is a distributed, search-optimized open source packet capture system. It was designed to
replace expensive, commercial appliances with off-the-shelf hardware and a free, easy to manage 
software system. Captured packets are reorganized during capture by flow (an indefinite length
sequence of packets with the same src/dst ips/ports and transport proto), indexed by flow, and
searched (again) by flow. The indexes for the captured packets are relatively tiny (typically less
than 1% the size of the captured data). 

For hardware requirements, see [HARDWARE.md](HARDWARE.md).

## DESTDIR
Many things in this file refer to DESTDIR as a pathname prefix. The default, and that used by future pcapdb packages, is `/var/pcapdb`.

## Architectural Overview 
A PcapDB installation consists of a Search Head and one or more Capture Nodes. The Search Head can
also be a Capture Node, or it can be a VM somewhere else. Wherever it is, the Search Head must be
accessible by the Capture Nodes, but there's no need for the Capture Nodes to be visible to the 
Search Head.

# Requirements 
PcapDB is designed to work on Linux servers only. It was developed on both Redhat Enterprise and
Debian systems, but its primary testbed has so far been Redhat based. While it has been verified to
work (with packages from non-default repositories) on RHEL 6, a more bleeding edge system (like
RHEL/Centos 7, or the latest Debian/Ubuntu LTS) will greatly simplify the process of gathering dependencies.

[sys_requirements.md](sys_requirements.md) contains a list of the packages required to run and build pcapdb.

requirements.txt contains python/pip requirements. They will be installed via 'make install'.

# Installing 
To build and install everything in /var/pcapdb/, run one of:
```
make install-search-head
make install-capture-node
make install-monolithic
```
 - Like with most Makefiles, you can set the DESTDIR environment variable to specify where to
   install the system. `make install-search-head DESTDIR=/var/mypcaplocation`
 - This includes installing in place: `make install-capture-node DESTDIR=$(pwd)`. In this case, PcapDB 
   won't install system scripts for various needed components, and it will run as the installing user.

To make your life easier, however, you should work make sure the indexing code builds cleanly by running 'make' in the 'indexer/' directory.

Postgresql may install in a strange location, as noted in the 'indexer/README'. This can cause build
failures in certain pip installed packages. Add `PATH=$PATH:<pgsql_bin_path>` to the end of your
'make install' command to fix this. For me, it is: `make install PATH=$PATH:/usr/pgsql-9.4/bin`.

# Setup 
After running 'make install', there are a few more steps to perform. 

Running from DESTDIR (your installation directory) `sudo core/bin/rabbitmq_setup.sh` will setup rabbitmq for use with pcapdb, create a password for the pcapdb account, and automatically set that password in the the pcapdb config file.

## DESTDIR/etc/pcapdb.cfg 
This is the main Pcapdb config file. You must set certain values before PcapDB will run at all. Those settings are noted in the file. 

## RabbitMQ 
RabbitMQ is a fast and efficient messaging system used to communicate simple messages between a
distributed network of hosts. As with Celery, RabbitMQ is really meant for distributing messages to
the 'first available' worker, but in PcapDB all of our messages are to a specific worker. As such,
the PcapDB rabbitMQ instance automatically creates a specific message queue for each Capture Node,
as well as a queue for the Search Head. The command 'rabbitmqctl' gives visibility into the
currently active queues. For further debugging/introspection, the rabbitmq admin plugin provides a
web interface that can be quite useful.

RabbitMQ server need only be configured on the search head. We've provided a script that does most of the
work:
```
$ /var/pcapdb/core/bin/rabbitmq_setup.sh
```

The above script will setup rabbitmq and a global user used by the search head and all capture
nodes. The login information is automatically populated in /var/pcapdb/etc/pcapdb.cfg on the search
head, but will need to be added to the pcapdb.cfg file for each Capture Node manually.

## Database Setup 
The setup varies significantly between the search head and capture nodes. 

### Add the 'capture' Role 
On all pcapdb servers, add a 'capture' role with login privileges and a password. As the postgres user:
```
sudo su -postgres
createuser capture -l -P
```

The 'db\_pass' variable in the pcapdb.cfg file should be set to the Search Head's db password on all
pcapdb hosts in the network. 

### On the Search Head 
Create a database named "pcapdb":
```
createdb -O capture pcapdb
```

Edit the Search Head's "pg\_hba.conf" (location varies) file to allow connections to Search Head DB from localhost. Also add a line allowing each capture node. 
```
host    pcapdb          capture         127.0.0.1/32            md5
host    pcapdb          capture         <capture node ip>       md5
```

Edit the Search Head's postgresql.conf file so that it listens on it's own IP:
```
listen_addresses = 'localhost,<search head ip>'
```

### On the Capture Nodes 
Create a database named 'capture\_node' on each capture node host:
```
createdb -O capture capture_node
```

Since the capture nodes connect via peer (unix socket) to their own database, no additional setup
should be needed.


### Restart the postgresql server.
```
# On most systems...
service postgresql restart
```

### Install the Database Tables 
After restarting the postgres service, we'll need to install the database tables on each 
PcapDB host. From the PcapDB installation directory (typically /var/pcapdb):
```
sudo su - root
cd /var/pcapdb
./bin/python core/manage.py makemigrations stats_api login_api core task_api search_head_api
login_gui search_head_gui capture_node_api

# On the search head
./bin/python core/manage.py migrate

# On the capture nodes
./bin/python core/manage.py migrate --database=capture_node
```

## Web server setup 
The Makefile will generate a self-signed cert for your server for you if an installed one doesn't
already exist at '/etc/ssl/<HOSTNAME>.pem'. You should probably change that to something that isn't
self-signed.

## Firewall Notes 
 - The Capture Nodes don't open any incoming ports for PcapDB, all communication is out to the Search Head.
 - Using IPtables or other system firewalls on the Capture Nodes is discouraged. Instead
   put them on a normally inaccessible network. They shouldn't generally have any ports open other than ssh.
 - The Search Head needs to be accessible by the Capture Nodes on ports 443, 5432 (postgres), and
   25672 (rabbitmq). 
   - It's ok to us IP tables on the Search Head (and will eventually be automatic)

# Running the system 
If you installed anywhere except 'in place', the system should attempt to run itself via
supervisord. __You'll have to restart some processes, as supervisord will have given up on them.__
 - The `supervisorctl` command can give you the status of the various components of the system. Capture has to be started manually from within the interface, so you shouldn't expect it to be running initially.
 - The `capture_runner` process, however, should be running. From within supervisorctl, 

 - The `core/runserver` and `core/runcelery` scripts will be helpful when not running the system in
   production.
 - Similarly, to run capture outside of production, use the capture_runner.py script: 
   `DESTDIR/bin/python DESTDIR/core/bin/capture_runner.py`
   - DESTDIR/log/django.log will tell you the exact command used to start capture, if for some reason it's failing to start.
   - DESTDIR/log/capture.log will usually give you some idea why capture is failing to run. If this file doesn't exist, either capture has never successfully ran at all, or rsyslog isn't forwarding the logs to the right place.

## System Component Hierarchy 
The PcapDB Search Head install consists of a PostgreSQL server, a Celery task queueing system, a
RabbitMQ messaging system, a uWSGI service, an NGINX webserver, and SupervisorD process management
system. The Capture Nodes are simpler, running only Celery, PostgreSQL SupervisorD, and the PcapDB
capture process. 

### PostgreSQL 
While PcapDB is a database of packets, it uses postgres to take care of more mundane database tasks.
The Search Head has a unique database that houses information on users, capture nodes, celery
response data, and aggregate statistics for the entire network of PcapDB capture nodes. All PcapDB
hosts must be able to connect to the Search Head database. 

Each Capture Node also has a database that keeps track of the available disk chunks and indexes on
that node. This database is only accessible to the capture node itself.

This has to be set up manually. See above for more information.

### Celery 
Celery is a system for distributing and scheduling tasks across a network of workers. PcapDB manages
all of it's communications with the Capture Nodes through Celery tasks, from initiating searches to
managing disk arrays. The tasks are assigned and picked up by the appropriate host via RabbitMQ
messaging queues, and the responses are saved to the search head via the search head's database.
Celery runs on both the Search Head and all Capture Nodes, though each host subscribes to different
task queues (see RabbitMQ below). 

Celery is configured automatically on system install, and the process is managed via supervisord.


### uWSGI and Nginx 
The web interface for PcapDB is built in the Python Django system, which is served via a unix
socket using uWSGI and persistant Python instances. Nginx handles all of the standard HTTP/HTTPS
portions of the web service, and passes Django requests to uWSGI via it's socket. (This is a pretty
standard way of doing things). 

 - uWSGI and Nginx are automatically configured on install.
 - Nginx is managed as a standard system service.
 - uWSGI is managed via supervisord

#### Certificates 
The standard configuration for PcapDB and Nginx expects ssl certificates and a private key installed
at:
```
/etc/ssl/<HOSTNAME>.pem
/etc/ssl/<HOSTNAME>.prv
```

If these don't already exist when running make install, self-signed certs will be created
automatically (with your input).


# A Few Other Tasks

## Install static files
You will need to install the static files for pcapdb. 

```
sudo su - capture
./bin/python core/manage.py collectstatic
```
# Making sure everything is working. 
After installing and setting up the database, there are a few things you can check to make sure
everything is working. 

 1. Restart supervisord to reset the uwsgi and celery processes to pick up the new database configs.

 2. Go to your webserver's root directory https://<myserver>/, and you should get to the pcapdb
 login page.
