
# Pgpool 

### What is Pgpool ?

Pgpool is a middleware for Postgre that sits between the client and the database server. It provides several key features to enhance a Postgre database's performance, high availability, and scalability. It is not a replacement for Postgres.

Some key benefits of Pgpool include ….

Connection Pooling which is one of the most fundamental features. Instead of creating a new connection for every client request, Pgpool maintains a pool of open connections to the Postgres server. When a new client request comes in, Pgpool can reuse an existing connection from the pool. This significantly reduces the overhead of establishing new connections, which is particularly beneficial for applications with frequent, short lived connections.

Pgpool can manage multiple Postgres servers, including a primary server and one or more replica servers. It can intelligently distribute read queries among the replica servers to spread the load and improve performance. This is known as read write splitting. Write queries are always sent to the primary server to maintain data consistency.

By using a primary and replica setup, Pgpool provides high availability. If the primary server fails, Pgpool can automatically detect the failure and promote a replica to become the new primary.  We will cover this in the tutorial.

Pgpool can also cache the results of frequently executed SELECT queries. If the same query is requested again, it can serve the result directly from its cache instead of executing the query on the database server. This can dramatically improve the response time for read heavy workloads.

Obviously, these are all options which can be left out or included in your implementation.

### Comparing Pgpool to Patroni or Pgbouncer

Pgpool's place in the Postgres community is best understood as a multi featured tool that is more complex than a single purpose utility like PgBouncer, but significantly less complicated to operate than a full fledged cluster manager like Patroni.

This middle ground is what makes it an attractive alternative for many use cases.

#### Pgpool vs Patroni

The core reason Pgpool is a less complicated alternative to Patroni is that it operates as a passive proxy rather than an active cluster manager. While Patroni is a powerful tool for building a robust high availability system, its power comes from a more complex design with more components to manage and worry about.

* Patroni’s entire design relies on a distributed consensus store like etcd or consul. This store is the single source of truth for the cluster's state, tracking which node is the primary and which are the replicas. While this model is highly resilient, it introduces a **significant** layer of operational complexity. You now have a third, critical piece of infrastructure to manage and monitor in addition to your Postgres instances. Pgpool, on the other hand, uses its own internal watchdog process to monitor node health and handle failover, eliminating the need for an external, third party component.  
* Passive vs. Active Control is the most crucial distinction between Pgpool and Patroni. Patroni actively manages the lifecycle of your Postgres instances. It starts and stops them, performs promote commands on replicas, and modifies their configurations based on the state in etcd. It essentially **takes over** the management of your Postgres cluster. Pgpool, on the other hand, is a simple proxy that sits in front of your database servers. It monitors their health and routes queries, but it doesn't control the Postgres processes themselves. It doesn't tell a replica to become a primary unless you configure it to do so. It simply detects that one has become a primary and starts routing traffic to it. This hands off approach makes it much simpler to integrate into existing database environments without rearchitecting your entire system.

#### Pgpool vs PgBouncer

The comparison with PgBouncer is different, as PgBouncer is a more narrowly focused tool.

* PgBouncer is designed to do one thing: connection pooling. It is a lightweight and highly efficient tool for handling a large number of client connections by reusing a smaller pool of connections to the Postgres server.  
* PgBouncer does not offer other features like load balancing, automatic failover, query routing or query caching. Its singular purpose makes it somewhat simple and fast. Pgpool, in contrast, is a multi purpose tool that bundles connection pooling, load balancing, automatic failover, and query caching into a single package.

Because it does more, Pgpool is slightly more complex to configure than PgBouncer, though it provides a more comprehensive set of high availability features with granular control of what features to make use of.

### Read this if you are not a patient person

This documentation provides both in depth explanations of key Postgres and Pgpool components and concise, minimal instructions for setting up your environment. If you want to get straight to the point, you can skip the detailed sections and jump directly to the highlighted code blocks

### Getting started

The process outlined in this documentation will guide you through the implementation of a robust Postgres database system utilizing Pgpool for both load balancing and automatic failover. 

To simplify the setup outlined here, I have created a custom Docker environment that comes pre packaged with Postgres 17 and many of the industry standard tools used with Postgres. The Docker files to create your local docker image can be downloaded directly from the following Git repository at 

**https://github.com/jtorral/rocky9-pg17-bundle.**

### 1. Build the Docker image

To create the Docker image, clone the repository from the provided GitHub link and run the docker build command. This command builds an image tagged as **rocky9-pg17-bundle** from the Dockerfile in the local directory.

    docker build \-t rocky9-pg17-bundle . 

### 2. Create the containers

For our environment, we will create 4 Postgres database servers and one server for Pgpool

First, create a network for the containers.

    docker network create pgnet 


Now, create the 4 postgres containers

    docker run -p 6431:5432 --env=PGPASSWORD=postgres -v pg1-pgdata:/pgdata --hostname pg1 --network=pgnet --name=pg1 -dt rocky9-pg17-bundle
    
    docker run -p 6432:5432 --env=PGPASSWORD=postgres -v pg2-pgdata:/pgdata --hostname pg2 --network=pgnet --name=pg2 -dt rocky9-pg17-bundle
    
    docker run -p 6433:5432 --env=PGPASSWORD=postgres -v pg3-pgdata:/pgdata --hostname pg3 --network=pgnet --name=pg3 -dt rocky9-pg17-bundle
    
    docker run -p 6434:5432 --env=PGPASSWORD=postgres -v pg4-pgdata:/pgdata --hostname pg4 --network=pgnet --name=pg4 -dt rocky9-pg17-bundle




**Lastly, create the pgpool container**

    docker run -p 7432:5432 -p 9999:9999 -p 9898:9898 --env=PGPASSWORD=postgres -v pgpool-pgdata:/pgdata --hostname pgpool  --network=pgnet --name=pgpool -dt rocky9-pg17-bundle 

Just like in the Postgres containers, we map specific ports that Pgpool uses, specifically, 9898 and 9999\. We will get into more details about this later.

### 3. Prepare the Postgres servers

To streamline the setup, the Docker containers come pre configured with the Postgres database server and SSH enabled. This configuration provides the postgres user with passwordless connections between all containers, which is necessary for their communication and a requirement for some of the features we will use from Pgpool in our tutorial.

***Let me say right from the beginning, one of the most challenging aspects of Pgpool is understanding the authentication process between user, client, server, admin and anything else that interacts with Pgpool. Let me just say, it is a nightmare until you understand what is actually happening. There is no consistency to the way Pgpool uses authentication.***  

Keep in mind, the containers used here are packaged with a variety of available Postgres tools. Therefore, we will need to make some slight modifications to meet the requirements of Pgpool.

**Lets log into the pg1 container.**

    docker exec \-it pg1 /bin/bash

Then su to postgres

    su - postgres

Make sure your environment is set just in case something went wrong with the container build. 

    echo $PGDATA

Should generate the following

    /pgdata/17/data

If you run the above echo **$PGDATA** command and the output is **/pgdata/17/data**  we are good to go.

    pg_ctl start

Create some test accounts and a needed extension in Postgres so we can start using Pgpool.

Start a psql session on **pg1**

    [postgres@pg1 ~]$ psql
    psql (17.6)
    Type "help" for help.
    
    postgres=#


**Create the following roles** 

    create role pgpool with login password 'pgpool';
    create role reader with login password 'reader';
    create role writer with login password 'writer';
    create role foobar with login password 'foobar';
    create role replicator with replication login password 'replicator'; 


Pgpool needs the extension **pgpool\_recovery** created in the template1 database in order to run certain commands against the database.

Connect to the template1 database and create the extension.

    \c template1

Create the necessary extension

    create extension pgpool_recovery;

Validate the extension was created.

    template1=#\dx
                                List of installed extensions
          Name       | Version |   Schema   |                Description
    -----------------+---------+------------+-------------------------------------------
     pgpool_recovery | 1.4     | public     | recovery functions for pgpool-II for V4.3
     plpgsql         | 1.0     | pg_catalog | PL/pgSQL procedural language 
    (2 rows)


### 4. Prepare the Pgpool server

As previously discussed, we'll now shift our focus to the dedicated Pgpool server. This machine will serve as the central point for managing our Postgres cluster's connections and traffic. The next steps involve configuring this server to enable its core functionalities and its command line tools.

**A rant about authentication in Pgpool.** 

Authentication in Pgpool can be a frustrating puzzle, a testament to its evolution rather than a cohesive design. It's a journey where you feel like you're learning a new secret handshake for every door you want to open.

Consider the user pcpadmin. To authenticate with Pgpool's PCP interface, you must define their password as an MD5 hash in the pcp.conf file. But when you run a PCP command like pcp\_node\_info, the client utility expects a plain-text password from a .pcppass file. The two pieces only fit together if one is the MD5 representation of the other.

Then there’s the recovery\_user in pgpool.conf, the one that needs to talk to the backend Postgres servers. You'd think it would follow a similar rule, but no. The recovery\_password cannot be an MD5 hash. It must be plain-text or AES-256 encrypted. So, while your PCP users are using MD5, your recovery users are on a different protocol entirely.

And the inconsistency doesn't stop there. When pg\_basebackup runs as part of the recovery process, it uses the password from the recovery\_password parameter in pgpool.conf (or a .pgpass file) to connect to the backend. This password must be a plain-text password that the backend Postgres server will accept, not a hash. It’s a constant dance between different hash formats, plain text, and encryption methods, all for a single piece of software. It's not about finding one correct method; it's about knowing which of the three or four correct methods applies to the specific action you want to perform.

The good news is that once it is set up and running, you are on solid ground.

So, with my rant about authentication out of the way, let's move on.

#### PCP tools

An essential part of this configuration is setting up the PCP (Pgpool Control Protocol) tools. These are a set of command line utilities that allow us to interact with and manage the Pgpool instance directly. Rather than editing configuration files and restarting the service, For example, we can use PCP tools to perform administrative tasks such as:

* Checking the status of each Postgres backend node.  
* Attaching or detaching a node from the cluster.  
* Promoting a replica to become the new primary server.  
* Displaying health check statistics.  
* Reloading the Pgpool configuration without stopping the service.

When you authenticate with PCP tools, the authentication method is completely separate from the regular Postgres user accounts. 

PCP uses its own, independent authentication mechanism defined in Pgpool's configuration file, typically **`pcp.conf`**. Instead of relying on usernames and passwords stored within the Postgres database, you create a dedicated user and a hashed password specifically for administering the Pgpool instance. This means that a user account used to manage Pgpool, does not have to be a valid user within the Postgres database itself. 

**Note** : Keep in mind, Pgpool config files are typically located in the **/etc/pgpool-II/** directory.

### Setting up PCP


#### pcp.conf

### Open up another terminal session and log on to the docker container.

    docker exec \-it pgpool /bin/bash

sudo to postgres

    su - postgres

Now cd to the pgpool directory

    cd /etc/pgpool-II

The **pcp.conf** file has the credentials for the user stored in the following format 

    <username>:<md5 password hash>

For this tutorial the username will be **pcpadmin** and the password will be **pcpadmin**.

From the command line run the following which will generate the md5 hash for **“pcpadmin”**

    pg_md5 pcpadmin

which will generate the following

    3090be164580c74a25efe7abdf542eb2

Copy the hash generated above.

    vi pcp.conf

Clear out the existing content and add the following line to the end of the file.

    pcpadmin:3090be164580c74a25efe7abdf542eb2

Save the file and make sure the permissions are set to 600

    chmod 600 pcp.conf

#### .pcppass

The **`.pcppass`** file is an optional, client side file used to store the credentials for connecting to the Pgpool instance using the PCP tools. Instead of typing the password every time you run a command, the tool can read it from this file. 

The file is located in the user's home directory and must have strict permissions of 0600  to protect the credentials. Since we are going to be using the **postgres** account to run most of the pgpool commands in our instance, lets set up the .pcppass file for the user postgres.

The format is simple: **hostname:port:user:password**

The password will be in plain text.


    cd \~

Create the .pcpass file

    vi .pcppass

Add the following two lines

    localhost:9898:pcpadmin:pcpadmin
    pgpool:9898:pcpadmin:pcpadmin

Save your changes and modify the permissions to 600

    chmod 600 .pcppass

Port 9898 is the port that pcp is listening on.

It's crucial to remember that the credentials in this file are for logging into and managing **Pgpool** itself, not for connecting to the database. 

#### pool_passwd

The pgpool_passwd file is a critical configuration file used by Pgpool to securely store the encrypted passwords for connecting to the backend Postgres database servers. Pgpool uses these credentials to authenticate itself to the primary and replica nodes, which is necessary for core features like replication and failover to function correctly. This file ensures that Pgpool can securely manage its connections to the databases without storing plaintext passwords.

For our install, we are using AES encryption and the pool\_passwd file needs to reflect that.

#### .pgpoolkey

The .pgpoolkey file is an optional file used to securely store a password for AES-256 encryption. It's a plain-text file that contains the encryption key, which is used by the pg\_enc utility to encrypt and decrypt passwords. Since we are using AES encryption, we will need the .pgpoolkey file for our install.

For security, the **.pgpoolkey** file must be located in the home directory of the user running Pgpool and must have its permissions set to 600\.

Since we are running Pgpool under the user postgres the .pgpass must be in the postgres home directory. In our environment, that would be **/var/lib/pgsql**

Assuming you have already su \- postgres as mentioned above, run the following commands.

    echo 'jorgeiscool' > ~/.pgpoolkey

Set the permissions

    chmod 600 ~/.pgpoolkey

### Generate entries for pool_passwd

When you run the following command for each user, you will be prompted for a password, after you enter the password, all the voodo takes place and the entry will be added to the pool\_pwasswd file.

For example, you will see a similar output to this.

    pg_enc -m -k ~/.pgpoolkey -u postgres -p
    
    db password:
    
    trying to read key from file /var/lib/pgsql/.pgpoolkey 


Remember to have your .**pgpoolkey** already in place and run the pg\_enc command below for each of the users created in the pg1 database.

The postgres user

    pg_enc -m -k ~/.pgpoolkey -u postgres -p 

The pgpool user

    pg_enc -m -k ~/.pgpoolkey -u pgpool -p 

The reader user

    pg_enc -m -k ~/.pgpoolkey -u reader -p 

The writer user

    pg_enc -m -k ~/.pgpoolkey -u writer -p 

And finally the replicator user

    pg_enc -m -k ~/.pgpoolkey -u replicator -p 

Based on the role names and password we created earlier  in postgres, this is what our file  **/etc/pgpool-II/pool_passwd** should look like after we generate all the entries.

    postgres:AESFK+C+SsZCGWyT3Ms4q1MOg==
    pgpool:AESQ5bnrN9tu6V2y6NVoE1Pwg==
    reader:AESKo9PIJYuOa9kA/hClNKnUg==
    writer:AES1xhIjMkStiINppUoRE4mPA==
    replicator:AEStWQQrGkATgObEU1EymwlRQ==


#### .pgpass

The .**pgpass** file is a file used by Postgres client applications, including tools like psql and pg\_basebackup, to store passwords for connecting to a Postgres database. It works with Pgpool by providing the password needed to connect to the backend database nodes.

The .pgpass file is designed to automate the password entry process. When a Postgres client application tries to connect to a server, it checks for a .pgpass file in the user's home directory. It reads the file line by line, looking for an entry that matches the connection details:

* Hostname  
* Port  
* Database name  
* Username  
* Password

If a line matches all four criteria, the client automatically uses the plain-text password from that line and sends it to the server for authentication. This allows you to run scripts and commands without being manually prompted for a password.

When it comes to Pgpool, the .pgpass file is most crucial for the recovery process. When pcp\_recovery\_node is executed, Pgpool runs a recovery\_1st\_stage\_command (usually pg\_basebackup) on a backend node. This command needs a password to connect to the primary Postgres database, and it will look for that password in a .pgpass file located on the backend server, in the home directory of the user running the command.

*On a side note, this is where those authentication inconsistencies I mentioned earlier come into play. For example, a **replication_user** defined in pgpool.conf cannot use md5 for password. So, if your database is set to use md5 and you have the user defined in .pgpass, you will get an error.  The only workaround is to use AES encryption or put the password in plain text in the config file. However, for this install, thi sis not an issue since we are using AES encryption.*

#### Create the .pgpass file in the postgres home directory

    cd \~ 

Then 

    vi .pgpass 

Add the following to it.

    pg1:5432:*:postgres:postgres
    pg1:5432:*:replicator:replicator
    pg2:5432:*:postgres:postgres
    pg2:5432:*:replicator:replicator
    pg3:5432:*:postgres:postgres
    pg3:5432:*:replicator:replicator
    pg4:5432:*:postgres:postgres
    pg4:5432:*:replicator:replicator
    pgpool:9999:*:postgres:postgres

Save the changes

    chmod 600 .pgpass

The above plain text passwords in pgpass are the passwords we created for the postgres database users earlier in pg1.

Copy the new .pgpass file to all the database servers we created, pg1, pg2, pg3 and pg4.

    scp .pgpass pg1:/var/lib/pgsql/
    scp .pgpass pg2:/var/lib/pgsql/
    scp .pgpass pg3:/var/lib/pgsql/
    scp .pgpass pg4:/var/lib/pgsql/

                           


#### pgpool.conf

Now lets turn our focus to the **pgpool.conf** file located in **/etc/pgpool-II**

This can be a ridiculously large file due to the many configuration options Pgpool offers. However, our attempt here is to keep it simple based on the features we are using for our install.

For the set up we are working with,  the pgpool.conf below is what we will use.


#### Create the pgpool.conf file

    cd /etc/pgpool-II

Then 
 

    vi pgpool.conf 

Replace everything in the current file with the content below

    pool_passwd     = '/etc/pgpool-II/pool_passwd'
    
    # --- Connection Settings ---
    
    listen_addresses     = '*'
    port                 = 9999
    
    # --- pcp connection Settings ---
    
    pcp_socket_dir       = '/tmp'
    pcp_listen_addresses = '*'
    pcp_port             = 9898
    
    # --- Replication stuff
    
    backend_clustering_mode = 'streaming_replication'
    sr_check_period         = 10
    sr_check_user           = 'replicator'
    sr_check_password       = ''
    follow_primary_command  = '/etc/pgpool-II/follow_primary.sh %d %h %p %D %m %H %M %P %r %R'
    replication_mode        = off  # On means pgpool does the syncing
    master_slave_mode       = on   # On means postgres does the syncing.
    
    # --- Load Balancing and Query Routing ---
    
    load_balance_mode           = on
    reset_query_on_pool_release = on
    replicate_on_reset          = on
    
    # --- Backend Server Configuration ---
    
    backend_hostname0         = 'pg1'
    backend_port0             = 5432
    backend_weight0           = 1
    backend_data_directory0   = '/pgdata/17/data'
    backend_clustering_mode   = 'streaming_replication'
    backend_flag0             = 'ALLOW_TO_FAILOVER'
    backend_application_name0 = 'pgserver0'
    
    backend_hostname1         = 'pg2'
    backend_port1             = 5432
    backend_weight1           = 1
    backend_data_directory1   = '/pgdata/17/data'
    backend_clustering_mode   = 'streaming_replication'
    backend_flag1             = 'ALLOW_TO_FAILOVER'
    backend_application_name1 = 'pgserver1'
    
    backend_hostname2         = 'pg3'
    backend_port2             = 5432
    backend_weight2           = 1
    backend_data_directory2   = '/pgdata/17/data'
    backend_clustering_mode   = 'streaming_replication'
    backend_flag2             = 'ALLOW_TO_FAILOVER'
    backend_application_name2 = 'pgserver2'
    
    backend_hostname3         = 'pg4'
    backend_port3             = 5432
    backend_weight3           = 1
    backend_data_directory3   = '/pgdata/17/data'
    backend_clustering_mode   = 'streaming_replication'
    backend_flag3             = 'ALLOW_TO_FAILOVER'
    backend_application_name3 = 'pgserver3'
    
    # --- failover and recovery info
    
    online_recovery               = on
    detach_false_primary          = on
    failover_command              = '/etc/pgpool-II/failover.sh %d %h %p %D %m %H %M %P %r %R %N %S'
    recovery_user                 = 'postgres'
    recovery_password             = ''
    recovery_1st_stage_command    = 'recovery_1st_stage'
    recovery_timeout              = 60
    client_idle_limit_in_recovery = -1
    
    health_check_user             = 'pgpool'
    health_check_password         = ''
    health_check_database         = ''
    health_check_period           = 5
    health_check_timeout          = 30
    health_check_max_retries      = 5
    
    # --- disable cache
    statement_cache_mode = off
    query_cache_mode = off
    
    # --- logging info
    
    logging_collector = on
    #log_min_messages = DEBUG1
    log_destination = 'stderr'
    log_line_prefix = '%m: %a pid %p: '
    log_directory = '/var/log/pgpool_log'
    log_filename = 'pgpool-%a.log'
    log_truncate_on_rotation = on
    log_rotation_age = 1d
    log_rotation_size = 0
    log_per_node_statement = on
    notice_per_node_statement = on


Save the changes



### 5. Recovery and failover

What makes Pgpool such a great tool, is its ability to failover and recover from a failed primary to a standby replica. And this process is in your control.  You can create your own scripts which can be called on a primary down detection or for bringing up an old primary as a new replica. This flexibility gives you so much more control and understanding of the process vs black box fully managed instance by products such as Patroni.

With that said, we will be using the sample scripts that already come with Pgpool with some slight modifications.


#### Configure recovery and failover scripts

On the pgpool server which you should still be logged into,  there is the directory **/etc/pgpool-II/sample_scripts** which contains some sample scripts.

Copy the following files to the /etc/pgpool-II directory

    cp /etc/pgpool-II/sample_scripts/recovery_1st_stage.sample /etc/pgpool-II/recovery_1st_stage
    cp /etc/pgpool-II/sample_scripts/follow_primary.sh.sample /etc/pgpool-II/follow_primary.sh
    cp /etc/pgpool-II/sample_scripts/failover.sh.sample /etc/pgpool-II/failover.sh
    cp /etc/pgpool-II/sample_scripts/pgpool_remote_start.sample /etc/pgpool-II/pgpool_remote_start


#### Modify the recovery_1st_stage file in /etc/pgpool-II

Change the following variables in the file to reflect the values shown below: 

    REPLUSER=replicator

Remember,  these containers have the ssh keys in them already and they are named id_rsa 

    SSH_KEY_FILE=id_rsa

The above reflects the user we created for replication and the ssh key we have already installed as part of the docker container. 

Now find the following block

    ${PGHOME}/bin/pg_basebackup -h $PRIMARY_NODE_HOST -U $REPLUSER -p $PRIMARY_NODE_PORT -D $DEST_NODE_PGDATA -X stream

And add **–checkpoint=fast** to it like shown below

    ${PGHOME}/bin/pg_basebackup -h $PRIMARY_NODE_HOST -U $REPLUSER -p $PRIMARY_NODE_PORT -D $DEST_NODE_PGDATA --checkpoint=fast -X stream

This makes sure we do not wait for a checkpoint to take place, we are basically telling it to do a checkpoint at the time of executing the pg\_basebackup. 

Lastly, look for this block ( line number 56 if not close to it )


    cat > ${RECOVERYCONF} << EOT  
    primary_conninfo = 'host=${PRIMARY_NODE_HOST} port=${PRIMARY_NODE_PORT} user=${REPLUSER} application_name=${DEST_NODE_HOST} passfile=''/var/lib/pgsql/.pgpass'''  
    recovery_target_timeline = 'latest'  
    primary_slot_name = '${REPL_SLOT_NAME}'  
    EOT


And add the following below it

    # Compensate for postgresql.auto.conf having old connection string by removing entries from postgresql.auto.conf   
      
    sed -i -e '/^primary_conninfo/d' -e '/^primary_slot_name/d' -e '/^recovery_target_timeline/d' ${DEST_NODE_PGDATA}/postgresql.auto.conf


The above will address any stale connection string in the  **postgresql.auto.conf** if you setup replication manually without using the pcp tools.

#### Modify the follow_primary.sh file in /etc/pgpool-II

Change the following variables in the file to reflect the values shown below: 

    REPLUSER=replicator

The user for pcp

    PCP_USER=pcpadmin

And once again the ssh key

    SSH_KEY_FILE=id_rsa

Lastly, look for this block ( line number 142 if not close to it )


    cat > ${RECOVERYCONF} << EOT  
    primary_conninfo = 'host=${NEW_PRIMARY_NODE_HOST} port=${NEW_PRIMARY_NODE_PORT} user=${REPLUSER} application_name=${NODE_HOST} passfile=''/var/lib/pgsql/.pgpass'''  
    recovery_target_timeline = 'latest'  
    primary_slot_name = '${REPL_SLOT_NAME}'  
    EOT


And add the following block below it.

It's different from the first file you changed. So don’t just cut and paste


    # Compensate for postgresql.auto.conf having old connection string by removing entries from postgresql.auto.conf
         
    sed -i -e '/^primary_conninfo/d' -e '/^primary_slot_name/d' -e '/^recovery_target_timeline/d' ${NODE_PGDATA}/postgresql.auto.conf

#### Modify the failover.sh file in /etc/pgpool-II

Change the following variables in the file to reflect the values shown below: 

    SSH_KEY_FILE=id_rsa

#### Copy the following files to pg1 data directory

 - recovery_1st_stage 
 - pgpool_remote_start

    cd /etc/pgpool-II

Then just use scp

    scp recovery_1st_stage pg1:/pgdata/17/data/
    scp pgpool_remote_start pg1:/pgdata/17/data/

### 6. Ready to start up

With our changes in place, we should be good to start up now.

From inside the pgpool container as user postgres, start pgpool and set debug mode so we can see more details in the log file.

    pgpool -f /etc/pgpool-II/pgpool.conf 

Afterwards, you can cd **/var/log/pgpool** look for the log file and simply tail the file. 

    cd /car/log/pgpool_log

Find the latest log file then tail it.

    tail -f pgpool-Mon.log

You should see output similar to this

    .
    .
    .
    .
    2025-09-10 21:36:06.173: sr_check_worker pid 931: DEBUG:  verify_backend_node_status: there's no standby node
    2025-09-10 21:36:06.173: sr_check_worker pid 931: DEBUG:  node status[0]: 1
    2025-09-10 21:36:06.173: sr_check_worker pid 931: DEBUG:  node status[1]: 0
    2025-09-10 21:36:06.173: sr_check_worker pid 931: DEBUG:  node status[2]: 0
    2025-09-10 21:36:06.173: sr_check_worker pid 931: DEBUG:  node status[3]: 0
    2025-09-10 21:36:06.173: sr_check_worker pid 931: DEBUG:  pool_release_follow_primary_lock called

Run a few tests ..

From the pgpool server as user postgres, execute 

    pcp_node_info -h localhost -U pcpadmin --all | column -t

You should see something similar to this output

    pg1 5432 1 0.250000 waiting up primary primary 0 none none 2025-09-08 22:18:33
    pg2 5432 3 0.250000 down down standby unknown 0 none none 2025-09-08 22:18:38
    pg3 5432 3 0.250000 down down standby unknown 0 none none 2025-09-08 22:18:39
    pg4 5432 3 0.250000 down down standby unknown 0 none none 2025-09-08 22:18:38



As you can see, node 0. Which is pg1 is up but the others are down because we have not brought them up yet as replicas of pg1.

Now let’s create a database in pg1 so we can see it replicate when we bring the other servers up.

On pg1 server, run

    $ psql
    psql (17.6)
    Type "help" for help.

Then,

    create database foobar with owner foobar;

Then a simple \l should show us the new database

    postgres=# \l
                                                     List of databases
       Name    |  Owner   | Encoding  | Locale Provider | Collate | Ctype | Locale | ICU Rules |   Access privileges
    -----------+----------+-----------+-----------------+---------+-------+--------+-----------+-----------------------
     foobar    | postgres | SQL_ASCII | libc            | C       | C     |        |           |
     postgres  | postgres | SQL_ASCII | libc            | C       | C     |        |           |
     template0 | postgres | SQL_ASCII | libc            | C       | C     |        |           | =c/postgres          +
               |          |           |                 |         |       |        |           | postgres=CTc/postgres
     template1 | postgres | SQL_ASCII | libc            | C       | C     |        |           | =c/postgres          +
               |          |           |                 |         |       |        |           | postgres=CTc/postgres
    (4 rows)
    
    postgres=#

Lets bring up a replica using the pcp commands

Back on the pgpool server as user postgres run

    pcp_recovery_node -U pcpadmin -h localhost -n 1

We get the following output

    pcp_recovery_node -- Command Successful

As you can see, when running psql on pg2 now we see the database foobar is there as well.

    [postgres@pg2 ~]$ psql
    psql (17.6)
    Type "help" for help.

Then list the database

    postgres=# \l
                                                     List of databases
       Name    |  Owner   | Encoding  | Locale Provider | Collate | Ctype | Locale | ICU Rules |   Access privileges
    -----------+----------+-----------+-----------------+---------+-------+--------+-----------+-----------------------
     foobar    | postgres | SQL_ASCII | libc            | C       | C     |        |           |
     postgres  | postgres | SQL_ASCII | libc            | C       | C     |        |           |
     template0 | postgres | SQL_ASCII | libc            | C       | C     |        |           | =c/postgres          +
               |          |           |                 |         |       |        |           | postgres=CTc/postgres
     template1 | postgres | SQL_ASCII | libc            | C       | C     |        |           | =c/postgres          +
               |          |           |                 |         |       |        |           | postgres=CTc/postgres
    (4 rows)
    
    postgres=#

Now bring up the other two replicas, node2 and node3 like we did node1


### 7. The PCP suite of tools

The Pgpool Command Protocol AKA  PCP tools are a set of command line utilities for managing and monitoring a running Pgpool instance. They allow an administrator to perform tasks remotely.

The following is a breakdown of what each PCP command is used for

**Node and Cluster Management**

 - **pcp_attach_node**: This command attaches a specific backend Postgres
   node to the Pgpool-II pool. It makes the node available for
   connections, bringing it back into the cluster. 
  - **pcp_detach_node**: This
   command detaches a given backend node from Pgpool-II, taking it out
   of service. It's useful for maintenance, as it prevents new
   connections from being routed to that node. 
   - **pcp_promote_node**: This command promotes a standby Postgres node to become the new primary.
   It's used for manual failover or switchover operations.
   https://www.pgpool.net/docs/46/en/html/pcp-promote-node.html
   - **pcp_recovery_node**: This command recovers a detached or failed node. It automatically takes a base backup from the primary and sets up replication, reattaching the node as a standby.

**Monitoring and Status**

- **pcp_health_check_stats**: This tool displays detailed health check statistics for a given backend node, including the number of successful and failed checks.
- **pcp_node_count**: This command returns the total number of Postgres nodes configured in Pgpool's pgpool.conf file, regardless of their current status.
- **pcp_node_info**: This command provides detailed information about a specific node, including its status, role (primary or standby), and other relevant data.
- **pcp_pool_status**: This command displays the current values of various Pgpool-II parameters, as defined in pgpool.conf.
- **pcp_proc_count**: This tool displays the total number of child processes running for Pgpool-II.
- **pcp_proc_info**: This command gives detailed information on a specific Pgpool-II child process ID.
- **pcp_watchdog_info**: This command displays the status of the Pgpool-II watchdog, which is used to monitor the Pgpool cluster itself for high availability.

**Configuration and Service Control**

- **pcp_invalidate_query_cache**: This command flushes the contents of Pgpool's in-memory query cache, forcing all subsequent queries to be re-evaluated.
- **pcp_log_rotate**: This command triggers the rotation of Pgpool-II's log files without requiring a restart or SIGHUP signal.
- **pcp_reload_config**: This command reloads the pgpool.conf file, applying changes to parameters that can be altered without a restart.
- **pcp_stop_pgpool**: This command gracefully shuts down the Pgpool-II process.


#### pcp_node_info

Displays information about a given node id or all nodes.

    pcp_node_info -h localhost -U pcpadmin --all | column -t

Will output the following or similar

    pg1  5432  2  0.250000  up       up  primary  primary  0  none  none  2025-09-10  21:35:56
    pg2  5432  1  0.250000  waiting  up  standby  standby  0  none  none  2025-09-10  21:40:02
    pg3  5432  1  0.250000  waiting  up  standby  standby  0  none  none  2025-09-10  21:39:34
    pg4  5432  1  0.250000  waiting  up  standby  standby  0  none  none  2025-09-10  21:39:38

#### What the columns represent

 1. hostname 
 2. port number 
 3. status 
 4. load balance weight 
 5. status name 
 6. actual backend status (obtained using PQpingParams. Pgpool-II 4.3 or later) 
 7. backend role 
 8. actual backend role (obtained using pg_is_in_recovery. Pgpool-II 4.3 or later) 
 9. replication delay 
 10. replication state (taken from pg_stat_replication. Pgpool-II 4.1 or later)     
 11. sync replication state (taken from pg_stat_replication. Pgpool-II 4.1 or later) 
 12. last status change time

#### What the status means

0 - This state is only used during the initialization. PCP will never display it.
1 - Node is up. No connections yet.
2 - Node is up. Connections are pooled.
3 - Node is down.


#### pcp_promote_node

Promotes the specified node as the new main / primary server

It is important to know the options between a switchover and not as this has an impact on the promotion.

[Read details here from the docs](https://www.pgpool.net/docs/latest/en/html/pcp-promote-node.html)

In our case we are using the  **\-s** option for a complete switchover.

    pcp_promote_node -h localhost -U pcpadmin -n  1 -s

Then we check the node info

    pcp_node_info -h localhost -U pcpadmin --all | column -t

As you can see, **pg2** ( which is node 1 as defined in pgpool.con ) is now the new primary.

    pg1 5432 3 0.250000 down up standby standby 0 none none 2025-09-10 18:15:25
    pg2 5432 1 0.250000 waiting up primary primary 0 none none 2025-09-10 18:15:25
    pg3 5432 3 0.250000 down up standby standby 0 none none 2025-09-10 18:15:25
    pg4 5432 3 0.250000 down up standby standby 0 none none 2025-09-10 18:15:25


#### pcp_reload_config

Reloads the pgpool config file

### 8.  Using built in extensions for monitoring


In Pgpool, you can use the SHOW command from the psql command line to display configuration parameters. This is similar to how you would inspect parameters in Postgres.

The available SHOW commands are:

-  pool_nodes: This command is the most commonly used. It displays the current status of all backend Postgres nodes that are managed by Pgpool. The output includes information like the node ID, hostname, port, status, and role (primary or standby).
-  pool_health_check_stats: This command provides detailed statistics on Pgpool's health check process for the backend nodes. It shows how many times a check has been successful, how many have failed, and the last time a check was performed.    
-  pool_processes: This command lists all the current Pgpool child processes, including their process ID, current status, and which backend node they are connected to.    
-   pool_status: This command displays various status variables related to the Pgpool instance, such as the total number of connections, the number of active processes, and other performance metrics.
    

The following demonstrate some of these tools

First connect to pgpool on port 9999

    psql -p 9999 -U postgres -h pgpool

Then we can run a few checks ...

#### pool_nodes

    postgres=# show pool_nodes;
     node_id | hostname | port | status | pg_status | lb_weight |  role   | pg_role | select_cnt | load_balance_node | replication_delay | replication_state | replication_sync_state | last_status_change
    ---------+----------+------+--------+-----------+-----------+---------+---------+------------+-------------------+-------------------+-------------------+------------------------+---------------------
     0       | pg1      | 5432 | up     | up        | 0.250000  | primary | primary | 0          | true              | 0                 |                   |                        | 2025-09-10 21:35:56
     1       | pg2      | 5432 | up     | up        | 0.250000  | standby | standby | 0          | false             | 0                 |                   |                        | 2025-09-10 22:47:21
     2       | pg3      | 5432 | up     | up        | 0.250000  | standby | standby | 0          | false             | 0                 |                   |                        | 2025-09-10 22:47:21
     3       | pg4      | 5432 | up     | up        | 0.250000  | standby | standby | 0          | false             | 0                 |                   |                        | 2025-09-10 22:47:21
    (4 rows)


#### pool_status


    postgres=# show pool_status;
                     item                  |                              value                              |                                   description
    ---------------------------------------+-----------------------------------------------------------------+---------------------------------------------------------------------------------
     backend_clustering_mode               | 1                                                               | clustering mode
     listen_addresses                      | *                                                               | host name(s) or IP address(es) to listen on
     port                                  | 9999                                                            | pgpool accepting port number
     unix_socket_directories               | /tmp                                                            | pgpool socket directories
     unix_socket_group                     |                                                                 | owning user of the unix sockets
     unix_socket_permissions               | 0777                                                            | access permissions of the unix sockets.
     pcp_listen_addresses                  | *                                                               | host name(s) or IP address(es) for pcp process to listen on
     pcp_port                              | 9898                                                            | PCP port # to bind
     pcp_socket_dir                        | /tmp                                                            | PCP socket directory
     enable_pool_hba                       | 0                                                               | if true, use pool_hba.conf for client authentication
     pool_passwd                           | /etc/pgpool-II/pool_passwd                                      | file name of pool_passwd for md5 authentication
     authentication_timeout                | 60                                                              | maximum time in seconds to complete client authentication
     allow_clear_text_frontend_auth        | 0                                                               | allow to use clear text password auth when pool_passwd does not contain passwor
     ssl                                   | 0                                                               | SSL support
     ssl_key                               |                                                                 | SSL private key file
     ssl_cert                              |                                                                 | SSL public certificate file


#### pool_health_check_stats


    postgres=# show pool_health_check_stats;
     node_id | hostname | port | status |  role   | last_status_change  | total_count | success_count | fail_count | skip_count | retry_count | average_retry_count | max_retry_count | max_duration | min_duration | average_duration |  last_health_check  | last_successful_health_check | last_skip_health_check | last_failed_health_check
    ---------+----------+------+--------+---------+---------------------+-------------+---------------+------------+------------+-------------+---------------------+-----------------+--------------+--------------+------------------+---------------------+------------------------------+------------------------+--------------------------
     0       | pg1      | 5432 | up     | primary | 2025-09-10 21:35:56 | 895         | 895           | 0          | 0          | 0           | 0.000000            | 0               | 16           | 5            | 8.098324         | 2025-09-10 22:50:33 | 2025-09-10 22:50:33          |                        |
     1       | pg2      | 5432 | up     | standby | 2025-09-10 22:47:21 | 852         | 846           | 0          | 6          | 0           | 0.000000            | 0               | 14           | 5            | 7.627660         | 2025-09-10 22:50:36 | 2025-09-10 22:50:36          | 2025-09-10 21:39:59    |
     2       | pg3      | 5432 | up     | standby | 2025-09-10 22:47:21 | 852         | 852           | 0          | 0          | 0           | 0.000000            | 0               | 13           | 5            | 7.536385         | 2025-09-10 22:50:36 | 2025-09-10 22:50:36          |                        |
     3       | pg4      | 5432 | up     | standby | 2025-09-10 22:47:21 | 852         | 851           | 0          | 1          | 0           | 0.000000            | 0               | 13           | 5            | 7.528790         | 2025-09-10 22:50:36 | 2025-09-10 22:50:36          | 2025-09-10 21:39:34    |
    (4 rows)



#### pool_processes

    postgres=# show pool_processes;
     pool_pid |     start_time      | client_connection_count | database | username | backend_connection_time | pool_counter |       status
    ----------+---------------------+-------------------------+----------+----------+-------------------------+--------------+---------------------
     1026     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1027     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1028     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1029     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1030     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1031     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1032     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1033     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1034     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1035     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1036     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1037     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1038     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1039     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1040     | 2025-09-10 21:40:03 | 0                       | postgres | postgres | 2025-09-10 22:47:21     | 1            | Execute command
     1041     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection
     1042     | 2025-09-10 21:40:03 | 0                       |          |          |                         |              | Wait for connection




### More to come as this is still work in progress. 
