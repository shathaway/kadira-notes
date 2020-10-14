# OSP Kadira APM
This is the Oregon State government implementation of the **kadira-open/kadira-server**.
OSP Kadira APM project creates a general purpose in-house
Meteor Application Performance Monitoring system.
The Meteor software development framework and the OSP Kadira APM are currently used 
for in-house software development products.
The OSP Kadira APM can be used to monitor any Meteor application
that is created or installed for in-house use.

Initial work is attributed to Arunoda Susiripala contributions to Kadira and Meteor.
He still claims to hold the Kadira name. Lots of related Kadira support has been
committed to github [kadirahq](https://github.com/kadirahq).

The hosting of OSP Kadira APM is targeted to a self-maintained server or network of servers.
The servers are either acquired as physical servers or PAAS leased platforms as a service.
Development of this implementation uses VMware virtual machines of Linux systems.
Development is compatible with a replicated set of mongod nodes on a single machine.

Our upgrade activites are based on **kadira-open/kadira-server** applications found in directories:
- kadira-engine
- kadira-rma
- kadira-ui

We are not doing deployments to cloud hosting services. All hosting is done in-house.

## Related services to support OSP Kadira APM include:

| MongoDb | A Mongo database implementing a replica set. |
| :-------------- | :---------------------------------------------------- |
| Kadira Services | These services can monitor Meteor instrumented applications. |
| NGINX | A web application proxy server supporting kadira-engine and kadira-ui. |
| systemd | A Linux system control manager. |

Strategic tasks for installing a new system:
- Install a MongoDb as a replica set.
- Install Kadira services as applications.
- Install a NGINX web service proxy for applications and services.
- Install Linux service controls for MongoDb, Kadira services, and NGINX.

## Installing MongoDb
A minimal installation of MongoDb is to provide two mongod services joined by a replica set.
If you only implement two mongod services, the primary service should have an
increased priority to resolve database replication node elections in absence of an arbiter.

If your implementation requires MongoDb coordination between hosts, then network security
should be implemented in your MongoDb configuration.
It is possible to implement a confined MongoDb database on a single platform with
no external network access and no configured security. You should ensure that your confined MongoDb uses only the loopback 127.0.0.1 interface for communications.

We currently install a self-managed MongoDb database on Linux with local hosting for our developers.
[MongoDB](https://docs.mongodb.com/manual/installation/) provides comprehensive installation guides.

Your installation may be different, but we use CentOS or RedHat hosting configured with SELinux security extensions. Our initial implementation uses localhost 127.0.0.1 binding with no external network access. This requires that all applications and database nodes be on the same host.

**mongod** - the primary database node implementation (default) : 27017
| /etc/mongod.conf | System startup options for primary node. |
| :--------------------------------- | :--------------------------- |
| /var/lib/mongod/* | Database for primary node. |
| /var/log/mongodb/mongod.log | Log file for primary node. |
| /lib/systemd/system/mongod.service | systemd unit description |
| /run/mongodb/mongod.pid | pid file for primary node |

**mondod2** - the secondary database node implementation (replica)  : 27020
| /etc/mongod2.conf | System startup options for secondary node.|
| :--------------------------------- | :--------------------------- |
| /var/lib/mongod2/* | Database for secondary node. |
| /var/log/mongodb/mongod2.log | Log file for secondary node. |
| /lib/systemd/system/mongod2.service | systemd unit description |
| /run/mongodb/mondod2.pid | pid file for secondary node |

**system files** - system management files and directories
| /etc/mongodb.d/* | Files for log file rotation |
| :--------------------------------- | :--------------------------- |
| /etc/mongodb.d/rotate.sh | Script to rotate log files |
| /etc/cron.d/mongodb.cron | Cron job policy for log rotation |
| /var/run/mongodb/* | pid files for mongod nodes |
| /var/log/mongodb/* | log files for mongod nodes |
| /lib/systemd/system/* | directory for systemd service files |

In our system **/var/run** is a soft link to **/run**.

### MongoDb Service Configuration Options

[MongoDb](http://docs.mongodb.org/manual/reference/configuration-options/) provides
detailed configuration configuration option descriptions.
Here are samples for our primary **mongod** and **mongod2** files of service options.

**First MongoDb Node (service options)**
```
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
storage:
  dbPath: /var/lib/mongod
  journal:
    enabled: true
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo
net:
  port: 27017
  bindIp: 127.0.0.1  # Listen to local interface only, comment to listen on all interfaces.
replication:
  oplogSizeMB: 160  # Initial oplog size in MB or 5% disk space
  replSetName: RS-Replica-01 # <string> multiple sets allowed
  enableMajorityReadConcern: false  # disable with 2-member mongod
```

**Second MongoDb Node (service options)**
```
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod2.log
storage:
  dbPath: /var/lib/mongod2
  journal:
    enabled: true
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod2.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo
net:
  port: 27020
  bindIp: 127.0.0.1  # Listen to local interface only, comment to listen on all interfaces.
replication:
  oplogSizeMB: 160  # Initial oplog size in MB or 5% disk space
  replSetName: RS-Replica-01 # <string> multiple sets allowed
  enableMajorityReadConcern: false  # disable with 2-member mongod
```

### Creating MongoDb ReplicaSet
After you create the primary **mongod** (port: 27017) and 
a standalone **mongod2** (port: 27020),
you then can bind them into a replicaSet using the Mongo shell
and initializing the replicaSet. Here is a sample replicaSet configuration.
The first node is given priority 2 to ensure it becomes the primary node.
Both nodes participate in service elections.

Let **RS-Replica-01** be the name for your replicaSet.

```js
{
  "_id" : "RS-Replica-01",
  "version" : 2,
  "protocolVersion" : NumberLong(1),
  "members" : [
    {
      "_id" : 0,
      "host" : "localhost:27017",
      "arbiterOnly" : false,
      "buildIndexes" : true,
      "hidden" : false,
      "priority" : 2,
      "tags" : {
      },
      "slaveDelay" : NumberLong(0),
      "votes" : 1
    },
    {
      "_id" : 1,
      "host" : "localhost:27020",
      "arbiterOnly" : false,
      "buildIndexes" : true,
      "hidden" : false,
      "priority" : 1,
      "tags" : {
      },
      "slaveDelay" : NumberLong(0),
      "votes" : 1
    }
  ],
  "settings" : {
    "chainingAllowed" : true,
    "heartbeatIntervalMillis" : 2000,
    "heartbeatTimeoutSecs" : 10,
    "electionTimeoutMillis" : 10000,
    "catchUpTimeoutMillis" : -1,
    "catchUpTakeoverDelayMillis" : 30000,
    "getLastErrorModes" : {
    },
    "getLastErrorDefaults" : {
      "w" : 1,
      "wtimeout" : 0
    }
  }
}
```

## Installing Prerequisites (NodeJs and Meteor)
Meteor provides its own compatible instance for **npm** and **node** commands.

A meteor bundle will use a **npm** and **node** outside of the **meteor**
environment for execution.

The **meteor** command is used for meteor project development and testing.

- **meteor-engine** is a node.js application project
- **meteor-npm** is a node.js application project
- **meteor-ui** is a meteor application project

The **node.js** projects require a usable instance of **npm** and **node** commands
outside of Meteor.  I like to use the **nvm** Node Version Manager to 
easily administer the version of the **node** environment to use.

### Installing NodeJs
The node.js runtime environment is required.
I like to use the (nvm) Node Version Manager so that it is easy to
switch between node.js and (npm) Node Package Manager releases.
This is useful for developer code maintence, but may cause issues
when deploying to a **kadira** account for running as a system sesrvice.

If your hosting is dedicated to a single version of npm and node.js,
you may wish to install a global execution path for npm and node.js.
Our development environment does not use global access for npm and node.js.
Versions managed by (nvm) and are cached in the user login directory
as the (~/.nvm) path.

### Installing Node Version Manager
The (nvm) Node Version Manager is available from [github.com/nvm-sh/nvm](https://github.com/nvm-sh/nvm).
Installing NVM is done by coping the "install.sh" file from the repository and running it from an account on your local machine.

~/.nvm/ is the directory that implements the **nvm** command and holds cached
executables for **npm** and **node** commands. The **.bashrc** bash configuration
file or its equivalent implements **nvm** as an alias command.

### Installing Meteor
See: [meteor.com/install](https://www.meteor.com/install) for installation notes.

This command downloads a shell script installation bundle.
```sh
curl https://install.meteor.com/ 
```
Installing this bundle puts the meteor-tool in your **~/.meteor/packages**
directory and installs a global launcher as executable file **/usr/local/bin/meteor**.

Each meteor project has a hidden **.meteor** directory containing the
meteor version to use with the project.
Invoking the **meteor** command from within a project chooses the proper
meteor-tool that provides the version specific runtime.
Meteor packages are cached in your login **~/.meteor/packages**
directory with references from your meteor project build directory.

You can run your meteor application with the meteor command,
or you can create an exportable bundle that can be run using a compatibe version of node.js.

After Meteor is installed to your account, the launching file **/usr/local/bin/meteor** 
should exist. A reference to directory **/usr/local/bin** should be in any
environment path that wishes to use the Meteor **meteor-tool** runtime.

## Installing OSP Kadira APM Applications
Here we assume you have a Linux user account.
It should not have the **kadira** name which will be described later
when we configure services.

You should copy the contents of the [shathaway/kadira-server](https://github.com/shathaway/kadira-server)
into a working directory of your choosing.
It is best to have a directory separate from that containing
the (*.git*) hidden directory that is associated with a forked or cloned git repository.

The find and cpio commands are useful for copying an entire directory structure to another.
Let SRC be the location of your working git repository.
Let DST be the location of your target path.
After copying, we are removing the .git files from the destination
because we don't need redundant git repositories.
```sh
cd $SRC
find | cpio -pdmuv $DST
cd $DST
rm -rf .git
```

The directory containing a copy of **kadira-server** content has a **init-shell.sh** file
that is used to set some common environment variables used by the application products
in the **kadira-engine**, **kadira-rma**, and **kadira-ui** directories.

### Create the Kadira Databases in MongoDb
This can be done using the MongoDb **mongo** shell.
There are two databases that need to be created.
These will be the names you use in the MongoDb connection strings used by the applications.
The database names have significance only in the connection URLs referenced by environment variables.
One database is the APP database containing the application and user registry.
The other database is the DATA database containing the application artifacts being measured.

#### Create the Databases and Connection Security
This example shows how to create the two databases and give them connection security
by using the MongoDb **mongo** shell.
Here the default localhost mongod server is specified,
but it can be any mongod server on which you have admistrative authority.

```
mongo localhost:27017
> use tkadira-data
> db.createUser({user: 'app', pwd: 'app-password', roles: [ 'readWrite', 'dbAdmin' ]})
> use tkadira-app
> db.createUser({user: 'app', pwd: 'app-password', roles: [ 'readWrite', 'dbAdmin' ]})
> show users
> exit
```

If your initial connection using mongo does not allow you to create databases, you will
need to supply administrative authentication credentials before creating users.
```
mongo mongo-host:port
> db.auth({user: 'system-user', pwd: 'system-password'})
```
If successful, you can then try to create the needed **APP** and **DATA** databases with
the connection authentication credentials.

The MongoDb database security used by the connection strings is unrelated to the Meteor user authentication mechanisms. The MongoDb security credentials are found in the "admin" database.
Here are examples we use in the **init-shell.sh** startup script.

Connecting to the APP database:
```
export APP\_MONGO_URL="mongodb://app:app-password@localhost:27017/tkadira-app"
```
The **app:app-password** represents the MongoDb security credentials associated with
the **tkadira-app** database.
```js
{ user: "app", pwd: "app-password" }
```

Connecting to the DATA database:
```
export DATA\_MONGO_URL="mongodb://app:app-password@localhost:27017/tkadira-data"
```
The **app:app-password** represents the MongoDb security credentials associated with
the **tkadira-app** database.
```js
{ user: "app", pwd: "app-password" }
```

Getting live statistics requires access to the Mongo OPLOG data stream
that synchronizes transactions to the replication cluster.
The OPLOG data stream uses the **local** database.
If your MongoDb system is secured, the user:password must have access to the local database,
but may be authenticated to a user on another database on the same server.
```sh
export APP\_MONGO\_OPLOG_URL="mongodb://app:app-password@localhost:27017,localhost:27020/local?repl
icaSet=RS-Replica-01&authSource=tkadira-app"
```

### Update the Startup Scripts

There are three ongoing applications as subdirectories to the overall architecture.

The top-level directory contains the "init-shell.sh" script that provides a common environment
to the related applications.

Each subordinate application (kadira-engine, kadira-rma, kadira-ui) have their own subdirectory
below the top-level. Each subdirectory has a "run.sh" responsible for starting that application.
To start a dependent application, you connect to the subdirectory and issue this command string.
```sh
cat ../init-shell.sh run.sh | sh
```
By this part of installation configuration, you should have enough information to
edit the "init-shell.sh" so the applications can be launched and testable.

Preparing the applications as Linux systemd services is described later.

### Initialize the DATA Database
Insert records into the mapReduceProfileConfig collection to support the metrics aggregation profiles.

This should only be done once for each shard. The default shard "one" is used for MongoDb installations 
that are not performing sharding on the DATA database.
```
use tkadira-data
db.mapReduceProfileConfig.insertMany([
{ "_id" : { "profile" : "1min", "provider" : "methods", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "1min", "provider" : "errors", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "1min", "provider" : "pubsub", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "1min", "provider" : "system", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "3hour", "provider" : "methods", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "3hour", "provider" : "errors", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "3hour", "provider" : "pubsub", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "3hour", "provider" : "system", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "30min", "provider" : "methods", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "30min", "provider" : "errors", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "30min", "provider" : "pubsub", "shard" : "one" }, "lastTime" : new Date() },
{ "_id" : { "profile" : "30min", "provider" : "system", "shard" : "one" }, "lastTime" : new Date() }
])
```

When doing sharding: "one" is the first shard, "two" is the second shard,
and "three" is the third shard.
Our developers are not doing sharding and therefore multiple
shard servers are not tested.

### Initialize APP Database With User
Our APM user authentication uses the Meteor **accounts-password** module.
The **kadira-ui** application uses the **APP\_MONGO_URL** environment for the
MongoDb connection to the our APP (tkadira_app) database.
APM user credentials are inserted into the **users** collection.
You can use the Meteor shell to install the initial user accounts.
The kadira-ui application must be running in order to launch the Meteor shell.

- Connect to the application project directory: kadira-ui
- Start the application: **cat ../init-shell.sh run.sh | sh &**

The application start above puts the job in the background. To cancel the application,
you bring it to the forground using the **fg 1** command and issue **<CTRL-C>**.
You can look for the background jobs by issuing the **jobs** command.
This will install a user with a basic account. The **plan** is *'free'* with no APM *'admin'* privileges.

At this time, the **kadira-ui** dashboard listening on *http://localhost:4000" is
unable to create new users.
New users are created with the Meteor shell.

The Meteor shell is invoked by a system user when connected to the application project directory
while the application is running. The database being used (*tkadira-app*) is the
application database.

```js
meteor shell
> Accounts.createUser({
... username: <string>,
... email: <string>,
... password: <string>
... })
 <returns _id: value>
> .exit
```
After creating a user, its login authorization should be available by using the dashboard.
The user has only default permissions associated with the "free" plan.

BEWARE: If you know the user document layout, you can use the **mongo** command to
access the database and directly administer the documents stored in collections.

## Testing the Kadira APM Applications

Here we launch the (kadira-engine), (kadira-rma), and (kadira-ui) as applications.

- kadira-engine -- This application receives metrics.
- kadira-rma -- This application performs mapReduce operations on metrics.
- kadira-ui -- This is the user dashboard to register applications and view metrics.

### Edit the "init-shell.sh" Environment
These are the serviceable environments provided by the original **kadira-open/kadira-server**
implementation. They should be edited for your environment.

- APP\_MONGO_URL -- The mongodb: connection string to your APP (tkadira-app) database.
- DATA\_MONGO_URL -- The mongodb: connection string to your DATA (tkadira-data) database.
- APP\_MONGO\_OPLOG_URL -- The mongodb: connection string to the OPLOG replicaSet synchronization stream.
- MAIL_URL -- The address of a **smtp** server to deliver email messages and notifications.
- ENGINE_PORT=11011 -- The port used by the **kadira-engine** listener service.
- UI_PORT=4000 -- The port used by the **kadira-ui** dashboard listener service.
- UI\_URL=http://localhost:$UI_PORT

We are not using the LIBRATO statics collection and reporting service.
The email interface and configured accounts are disabled or
removed from our code implementation.

We are not using the Amazon Web Services (AWS) or other cloud hosting services.

The (kadira-ui) application **settings.json** file is currently unchanged
from the initial **kadira-open** code base.


### Starting the Applications
The startup is consistent across all three applications (kadira-engine), (kadira-apm)
and (kadira-ui).

1. Connect to the application directory.
2. Invoke: `cat ../init-shell.sh run.sh | sh`

I like to change the second line to have redirection to a **runtime.log** file
that can be later reviewed for problems.

```sh
cat ../init-shell.sh run.sh | sh 2>&1 | tee runtime.log
```

Running the applications in the foreground, one application per terminal session,
will afford an easy application termination by issuing a (CTRL-C) or SIGHUP
to the controlling process.

Application **kadira-rma** launches a dozen node
processes that can be confusing to cleanup without knowing the controlling
process.

Knowledge of using the **ps** command to analyze Linux processes is useful.

Later we will configure (systemd/systemctl) to launch these applications
as services.

## Installing OSP Kadira APM Services (systemd/systemctl)

Services we implement for our OSP Kadira APM include:
- mongod
- mongod2
- kadira-en
- kadira-rma
- kadira-ui
- nginx

The **mongod** service is the default implemented by MongoDb installation.

The **mongod2** service is a clone of the configuration files with minor
edits to accommodate a second **mongod** service on the local machine.

The **kadira-en** service makes (systemd/systemctl) control an application
as a service. This service listens for activity on *"http://localhost:11011"*
with no external access except through an NGINX reverse proxy web server.

The **kadira-rma** service makes (systemd/systemctl) control an application
as a service.

The **kadira-ui** service makes (systemd/systemctl) control an application
as a service. This service listens for activity on *"http://localhost:4000"*
with no external access except through an NGINX reverse proxy web server.

The **nginx** service is the NGINX reverse proxy web server.

We have also implemented some NodeJs applications for testing. These normally
listen by default on *"http://localhost:3000"* which is default for NodeJs
applications.

We configure NGINX to publish external DNS names for our embargoed 
OSP Kadira APM applications.

- http://engine.example.com --> http://localhost:11011
- https://ui.example.com --> http://localhost:4000
- https://node.example.com --> http://localhost:3000

These applications provide extenal access through NGINX
to service locations differentiated by DNS name, and having a
common IP connection address.

There is a reason why "http:" is assiciated with *engine.example.com*.

The *mdg:meteor-apm-agent* and *meteorhacks:kadira* instrumentation for
Meteor applications have problems sending metrics to a secure "https:"
endpoint. An issue has been submitted to the Meteor Development Group
about this. Even though a secure "https:" connection can be
established from a Meteor client, the expected encrypted payload cannot
be recognized or even delivered to our NGINX reverse proxy web server.

The "kadira-rma" service runs locally with no external communications
except to a backend MongoDb database.

## MongoDb Services

The MonoDb mongod services are being run as user:mongod and group:mongod.

The log files are saved to the /var/log/mongodb/ directory.

The Linux pid files are: /var/run/mongod.pid and /var/run/mongod2.pid

The Linux (systemd/systemctl) service unit files are:<br>
/lib/systemd/system/mongod.service<br>
/lib/systemd/system/mongod2.service

The MongoDb **mongod** configuration files are in the /etc directory.
These files are used by the mongod instance being launched by the service manager.
See [MongoDb configuration options](https://docs.mongodb.org/manual/reference/configuration-options/)
for details.

File: /etc/mongod.conf<br>
This is the MongoDb configuration for the first **mongod** service.

File: /etc/mongod2.conf<br>
This is the MongoDb configuration for the second **mongod** service.

These files cause the **mongod** to run detached as a daemon and
provide a file that contains the Linux PID of the **mongod** process.
The Linux (systemd/systemctl) service manager uses these pid files
for controlling MongoDb services.

File: /lib/systemd/system/mongod.service
```
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network.target

[Service]
User=mongod
Group=mongod
Environment="OPTIONS=-f /etc/mongod.conf"
ExecStart=/usr/bin/mongod $OPTIONS
ExecStartPre=/usr/bin/mkdir -p /var/run/mongodb
ExecStartPre=/usr/bin/chown mongod:mongod /var/run/mongodb
ExecStartPre=/usr/bin/chmod 0755 /var/run/mongodb
PermissionsStartOnly=true
PIDFile=/var/run/mongodb/mongod.pid
Type=forking
\# file size
LimitFSIZE=infinity
\# cpu time
LimitCPU=infinity
\# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=64000
\# processes/threads
LimitNPROC=64000
\# locked memory
LimitMEMLOCK=infinity
\# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false
\# Recommended limits for for mongod as specified in
\# http://docs.mongodb.org/manual/reference/ulimit/#recommended-settings

[Install]
WantedBy=multi-user.target
```

File: mongod2.service differs only by these lines:
```
Environment="OPTIONS=-f /etc/mongod2.conf"
PIDFile=/var/run/mongodb/mongod2.pid
```

### Log File Rotation for Mongod Services
MongoDb can rotate the log files if the service process is
sent the SIGUSR1 signal.

A schedule can be installed as a cron job rule.

Here is a script that can be used to rotate log files.
```sh
\#!/bin/bash
\# File: /etc/mongodb.d/rotate.sh
\#
\# This file is called by /etc/cron.d/mongodb.cron
\#
\# Rotate the MongoDB replica set logs on localhost
\# We currently have 2 mongod processes on this host.
\#

PID1=0
PID2=0

\# get the PIDs of the running mongod services

if [ -f "/var/run/mongodb/mongod.pid" ] ; then
 PID1=\`cat /var/run/mongodb/mongod.pid`
fi
if [ -f "/var/run/mongodb/mongod2.pid" ] ; then
 PID2=\`cat /var/run/mongodb/mongod2.pid`
fi

\# Signal SIGUSR1 to rotate the mongod service logs

if [ ${PID1} -gt 1 ] ; then
 kill -s USR1 $PID1
fi
if [ ${PID2} -gt 1 ] ; then
 kill -s USR1 $PID2
fi
```
File: mongodb.cron
```
\# File: mongodb.cron
\#
\# Template to schedule the rotation of MongoDB log files.
\# Place this schedule into /etc/cron.d/mongodb.cron
\# (Minute 0,59) (hour 0,23) (day-of-month 1,31) (month-in-year 1,12) (day-of-week 0,6/0=sunday)

1 1 * * * /etc/mongodb.d/rotate.sh
```






