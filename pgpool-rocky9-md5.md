
# Pgpool basic setup with failover and load balancing

## Non docker (bare metal) with md5 authentication


This variation of the existing documentation for the pgpool setup and tutorial, is a direct evolution of the previous Docker based guides, but it's recontextualized for bare metal servers. While the core objective remains the same, installing and setting up Pgpool, this version will replace all container-specific instructions with methods native to the Linux operating system.

The key distinction is the management of the Pgpool process itself. In a Docker environment, the container orchestrator (Docker) handles the process lifecycle, user context, and networking.  On a bare metal server, the operating system's native service manager, **systemd**, is used to control Pgpool. This guide will include the use of systemd, and necessary unit file to start and stop Pgpool as the **postgres** user.



### Getting started


Again, This guide deviates from other documentation in this repository by focusing on a **bare metal** deployment rather than containerized environments like Docker. The instructions are tailored for servers running **Rocky Linux 9**, a derivative of Red Hat Enterprise Linux 9, and are intended for direct installation on the host operating system. If you are using a different OS, you may need to modify a few of the steps.

Additionally, this documentation references Postgres 17. If you are using an older version, please make note of any references to Postgres 17 and adjust as needed for your deployment.

### Prerequisites and Assumptions

The following technical prerequisites are assumed for the target audience of this document:

-   **Operating System:** The target servers are installed with a Red Hat 9 based operating system and are fully operational.
    
-   **Secure Shell (SSH):** The SSH service is installed and configured on all servers to facilitate remote access and management.
    
-   **SSH Key-Based Authentication:** All servers involved in the demonstration are configured for SSH key-based authentication to enable secure, password less communication between nodes. We will mention the accounts further on in the documentation.
    
-   **SELinux Configuration:** The Security-Enhanced Linux (SELinux) policy is configured to not interfere with the operation of Pgpool and the Postgres services. Especially ssh.
    
-   **Firewall Rules:** The host firewall (if runnin) is configured to allow bidirectional network traffic between the four servers essential for this demonstration.

- **Hostname Resolution:** The **/etc/hosts** file on each server has been updated to include the IP addresses and hostnames of all servers in the environment. This is crucial for enabling seamless hostname based communication among the cluster nodes.

**The servers for this deployment**

For this deployment, we will be using 4 separate servers. 3 for Postgres databases and 1 for the pgpool service.  The hosts name used for this will be **pg1, pg2, pg3** and **pgpool** 

With these components in place, we can proceed with the installation and configuration steps.




### Install Postgres and Pgpool on all servers

Although Pgpool is not required to reside on the database server, the database does need a specific Pgpool extension. I have found it convenient to just install Pgpool on the database servers and create the extension with the added bonus of being able to use the database servers for Pgpool as well in a redundant Pgpool deployment using watchdog.

As root, on all servers (**pg1, pg2, pg3 and pgpool** ) run the following.

    dnf update -y
    dnf install -y epel-release
    dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
    dnf -qy module disable postgresql
    dnf install -y postgresql17-server
    dnf install -y postgresql17-contrib
    dnf install -y https://www.pgpool.net/yum/rpms/4.6/redhat/rhel-9-x86_64/pgpool-II-release-4.6-1.noarch.rpm
    dnf config-manager --set-enabled crb
    dnf install -y libmemcached-awesome
    dnf install -y pgpool-II-pg17
    dnf install -y pgpool-II-pg17-extensions

**Note**

**If you are running Oracle 9 and you received the error, "nothing provides libmemcached.so.11()(64bit)" when trying to install Pgpool ... ,  you will probably need to do the following ...** 

Remember, the above instructions are based on **Rocky 9 OS**. 

The following is a light variation from the above references to **libmemcached** and **crb** to help resolve the issue. Simply run ...

    dnf config-manager --set-enabled ol9_codeready_builder
    dnf config-manager --set-enabled crb
    dnf install libmemcached-awesome libmemcached-awesome-devel

Then try to reinstall Pgpool.

    dnf install -y pgpool-II-pg17
    dnf install -y pgpool-II-pg17-extensions


The error  indicates that a dependency is missing.  Specifically, the package requires the **libmemcached.so.11** shared library, but your system can't find it in the enabled repositories. This is a common issue on RHEL-based systems like RHEL 9 when a package from one repository depends on a library that is in a different, un-enabled repository.

#### Reminder, Make sure the /etc/hosts file is updated with info for all servers (pg1, pg2, pg3 and pgpool).

#### Setup ssh for the postgres user across all servers

For this documentation, I simply generate an ssh key ( id_rsa) , add the public key (id_rsa.pub) to the (authorized_keys) file and copy them to the postgres users .ssh directory on all the servers.  This is not the most secure way of doing things but it makes things easier for this documentation.  As mentioned earlier, the assumption is made that you will take care of this as part of the install.

#### On the pgpool server (pgpool)

#### Setup a unit file for pgpool

If your install did not create the unit file for Pgpool, you will need to create it manually.  A quick way to see if the service is there is to run

    systemctl status pgpool.service

If you see see an output similar to the following, you are good to go.

  
    ○ pgpool.service - Pgpool-II
         Loaded: loaded (/usr/lib/systemd/system/pgpool.service; disabled; preset: disabled)
         Active: inactive (dead)


**If you get an error indicating the unit could not be found** 

    Unit pgpool.service could not be found.

Then perform the following steps

As user root

    vi /etc/systemd/system/pgpool.service

And add the following to the file.

    [Unit]
    Description=Pgpool-II
    After=syslog.target network.target
    
    [Service]
    
    User=postgres
    Group=postgres
    
    EnvironmentFile=-/etc/sysconfig/pgpool
    
    ExecStart=/usr/bin/pgpool -f /etc/pgpool-II/pgpool.conf $OPTS
    ExecStop=/usr/bin/pgpool -f /etc/pgpool-II/pgpool.conf $STOP_OPTS stop
    ExecReload=/usr/bin/pgpool -f /etc/pgpool-II/pgpool.conf reload
    
    [Install]
    WantedBy=multi-user.target

Now reload the daemon

    systemctl daemon-reload

**Perform the following regardless of whether you created the unit file or not.** 

One more thing ... 

    cp /usr/lib/tmpfiles.d/pgpool-II-pg17.conf /etc/tmpfiles.d

Now, lets set proper ownership where needed.

    cd /etc/
    chown -R postgres:postgres pgpool-II/
    
    cd /var/run
    chown -R postgres:postgres pgpool


### 3. Prepare the Postgres servers


#### On the pg1 database server (pg1) 


At this point Postgres should already be installed on all the servers.

    su - postgres

Then ...

    cd /var/lib/pgsql/17/data

Create the file **pg_custom.conf**

vi pg_custom.conf

Add the following to the file

    listen_addresses = '*'
    
    wal_level = 'logical'
    max_wal_size = 1GB
    wal_log_hints = on
    
    max_connections = 250
    hot_standby = on
    password_encryption = md5
    
    logging_collector = on
    log_filename = 'postgresql-%a.log'
    log_directory = log
    log_line_prefix = '%m [%r] [%p]: [%l-1] user=%u,db=%d,host=%h '
    log_checkpoints = on
    log_truncate_on_rotation = on
    log_lock_waits = on
    log_min_duration_statement = 500
    log_temp_files = 0
    log_autovacuum_min_duration = 0
    checkpoint_completion_target = 0.9

Save your changes.

The above config represents minimal configuration. You will need to adjust the runtime parameters to meet your specific needs.

Now modify the postgresql.conf file

    vi postgresql.conf

Add the following to the end of the file.

    include = 'pg_custom.conf'

Save your changes

Modify the **pg_hba.conf** file

    vi pg_hba.conf

Find and references to **scram-sha-256** or **trust**  and change them to md5.

The file should look like the this.

    local   all             postgres                                peer

    # TYPE  DATABASE        USER            ADDRESS                 METHOD

    # "local" is for Unix domain socket connections only
    local   all             all                                     peer


    # IPv4 local connections:
    host    all             all             127.0.0.1/32            md5
    host    all             all             0.0.0.0/0               md5

    # IPv6 local connections:
    host    all             all             ::1/128                 md5


    # Allow replication connections from localhost, by a user with the
    # replication privilege.
    local   replication     all                                     peer
    host    replication     all             127.0.0.1/32            md5
    host    replication     all             ::1/128                 md5
    host    replication     all             0.0.0.0/0               md5



**Start postgres**

    pg_ctl start

Create some test accounts and a needed extension in Postgres so we can start using Pgpool.

Start a psql session on **pg1**

    [postgres@pg1 ~]$ psql
    psql (17.6)
    Type "help" for help.

    postgres=#

Validate you are using md5 encryption.

    postgres=# show password_encryption;
     password_encryption
    ---------------------
     md5
    (1 row)

**
Create the following roles**

    create role pgpool with login password 'pgpool';
    create role reader with login password 'reader';
    create role writer with login password 'writer';
    create role foobar with login password 'foobar';
    create role replicator with replication login password 'replicator';

Now set the postgres password

    alter role postgres with password 'postgres';



Pgpool needs the extension **pgpool_recovery** created in the template1 database in order to run certain commands against the database.

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



### 4. Prepare the Pgpool server (pgpool)

As previously discussed, we'll now shift our focus to the dedicated Pgpool server. This machine will serve as the central point for managing our Postgres cluster's connections and traffic. The next steps involve configuring this server to enable its core functionalities and its command line tools.

### I need to say a few things about Pgpool and authentication.

Authentication in Pgpool can be a frustrating puzzle, a testament to its evolution rather than a cohesive design. It's a journey where you feel like you're learning a new secret handshake for every door you want to open.

Consider the user pcpadmin. To authenticate with Pgpool's PCP interface, you must define their password as an MD5 hash in the pcp.conf file. But when you run a PCP command like pcp_node_info, the client utility expects a plain-text password from a .pcppass file. The two pieces only fit together if one is the MD5 representation of the other.

Then there’s the recovery_user in pgpool.conf, the one that needs to talk to the backend Postgres servers. You'd think it would follow a similar rule, but no. The recovery_password cannot be an MD5 hash. It must be plain-text or AES-256 encrypted. So, while your PCP users are using MD5, your recovery users are on a different protocol entirely.

And the inconsistency doesn't stop there. When pg_basebackup runs as part of the recovery process, it uses the password from the recovery_password parameter in pgpool.conf (or a .pgpass file) to connect to the backend. This password must be a plain-text password that the backend Postgres server will accept, not a hash. It’s a constant dance between different hash formats, plain text, and encryption methods, all for a single piece of software. It's not about finding one correct method; it's about knowing which of the three or four correct methods applies to the specific action you want to perform.

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

#### Log on to the pgpool server (pgpool)

#### The pcp.conf file

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

#### The .pcppass file

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

#### The pool_passwd file

The pgpool_passwd file is a critical configuration file used by Pgpool to securely store the encrypted passwords for connecting to the backend Postgres database servers. Pgpool uses these credentials to authenticate itself to the primary and replica nodes, which is necessary for core features like replication and failover to function correctly. This file ensures that Pgpool can securely manage its connections to the databases without storing plaintext passwords.

### Generate entries for pool_passwd

When you run the following command for each user, an entry will be added to the **pool_passwd** file for the users.

**TIP:**

**If you are using md5 as in this documentation and have many users to add to the pool_passwd file, you could run the following query against your Postgres database which will generate the necessary output which you can cut and paste. Thus, saving you the time of generating a password for each user.**

    select usename || ':' ||  passwd from pg_shadow;

Otherwise, here is how it is performed using the pg_md5 tool.

The postgres user

    pg_md5 -m -u postgres postgres

The pgpool user

    pg_md5 -m -u pgpool pgpool

The reader user

    pg_md5 -m -u reader reader

The writer user

    pg_md5 -m -u writer writer

And finally the replicator user

    pg_md5 -m -u replicator replicator

If you have additional user you want to add, repeat the above command with the user and password.  In our examples, the password is the last argument of the command.

Based on the role names and password we created earlier  in Postgres, this is what our file  **/etc/pgpool-II/pool_passwd** should look like after we generate all the entries.

    postgres:md53175bce1d3201d16594cebf9d7eb3f9d
    pgpool:md5f24aeb1c3b7d05d7eaf2cd648c307092
    reader:md5594a23d93fbef31193790edc87763969
    writer:md5b462494acb3894fee8e680bb0057d319
    replicator:md55577127b7ffb431f05a1dcd318438d11


#### The .pgpass file

The .**pgpass** file is a file used by Postgres client applications, including tools like psql and pg_basebackup, to store passwords for connecting to a Postgres database. It works with Pgpool by providing the password needed to connect to the backend database nodes.

The .pgpass file is designed to automate the password entry process. When a Postgres client application tries to connect to a server, it checks for a .pgpass file in the user's home directory. It reads the file line by line, looking for an entry that matches the connection details:

* Hostname
* Port
* Database name
* Username
* Password

If a line matches all four criteria, the client automatically uses the plain-text password from that line and sends it to the server for authentication. This allows you to run scripts and commands without being manually prompted for a password.

When it comes to Pgpool, the .pgpass file is most crucial for the recovery process. When pcp_recovery_node is executed, Pgpool runs a recovery_1st_stage_command (usually pg_basebackup) on a backend node. This command needs a password to connect to the primary Postgres database, and it will look for that password in a .pgpass file located on the backend server, in the home directory of the user running the command.

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
 

#### The pgpool.conf file 

Now lets turn our focus to the **pgpool.conf** file located in **/etc/pgpool-II**

This can be a ridiculously large file due to the many configuration options Pgpool offers. However, our attempt here is to keep it simple based on the features we are using for our install.

For the set up we are working with,  the pgpool.conf below is what we will use.


#### Create the pgpool.conf file

    cd /etc/pgpool-II

Then


    vi pgpool.conf

Replace everything in the current file with the content below

    pool_passwd     = '/etc/pgpool-II/pool_passwd'
    auth_type       = 'md5'
    
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
    backend_data_directory0   = '/var/lib/pgsql/17/data'
    backend_clustering_mode   = 'streaming_replication'
    backend_flag0             = 'ALLOW_TO_FAILOVER'
    
    backend_hostname1         = 'pg2'
    backend_port1             = 5432
    backend_weight1           = 1
    backend_data_directory1   = '/var/lib/pgsql/17/data'
    backend_clustering_mode   = 'streaming_replication'
    backend_flag1             = 'ALLOW_TO_FAILOVER'
    
    backend_hostname2         = 'pg3'
    backend_port2             = 5432
    backend_weight2           = 1
    backend_data_directory2   = '/var/lib/pgsql/17/data'
    backend_clustering_mode   = 'streaming_replication'
    backend_flag2             = 'ALLOW_TO_FAILOVER'
    
    # --- failover and recovery info
    
    online_recovery               = on
    detach_false_primary          = on
    failover_command              = '/etc/pgpool-II/failover.sh %d %h %p %D %m %H %M %P %r %R %N %S'
    recovery_user                 = 'postgres'
    recovery_password             = 'postgres'
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
    log_min_messages = DEBUG1
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

### A special note about the recovery_user and recovery_passwd in pgpool.conf

When you are using md5 authentication with pgpool and postgres, the recovery_user and it's password are in the pgpool.conf file with the password defined in clear text.

    recovery_user                 = 'postgres'
    recovery_password             = 'postgres'

Under normal circumstances, other users and passwd settings can be left as an empty string '' which will make pgpool check for the password in the pool_passwd file and that password will be stored as an md5 hash in the pgpool_passwd file.

For example

    sr_check_user           = 'replicator'
    sr_check_password       = ''

However, **if you were using scram-sha-256 authentication**, the password for recovery_passwd can be an empty string.



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

This makes sure we do not wait for a checkpoint to take place, we are basically telling it to do a checkpoint at the time of executing the pg_basebackup.

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

With our changes in place, we should be good to start up Pgpool now.

We can manually start Pgpool for testing

    pgpool -f /etc/pgpool-II/pgpool.conf

Or as root, we can run

    systemctl start pgpool.service

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

In our case we are using the  **-s** option for a complete switchover.

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



### 9. Basic routing and load balancing

#### user_redirect_preference_list

Specifies the list of  _"user-name:node id(ratio)"_  pairs to send  SELECT  queries to a particular backend node for a particular user connection at a specified load balance ratio. The load balance ratio specifies a value between 0 and 1. The default is 1.0.

For example, by specifying "user1:1(0.5)",  Pgpool-II  will redirect 50%  SELECT  queries to the backend node of ID 1 for the connection of user  user1.

You can specify multiple  _"user-name:node id(ratio)"_  pairs by separating them using comma (,).

***A more simple approach to redirecting users especially in environments where the role of a node can change based on fail over scenarios, regular expressions are also accepted for user name.***

You can use special keywords. If  **"primary"**  is specified, queries are sent to the primary node, and if  **"standby"** is specified, one of the standby nodes are selected randomly based on load balance ratio.

#### user_redirect_preference_list = 'postgres:primary,reader:standby'

In the following example ...

    user_redirect_preference_list = 'postgres:primary,reader:standby'

When the user **postgres** connects to the database via pgpool, it will be directed to the primary server.
When the user **reader** connects to the database it will be routed to a standby server.

To achieve the above, we add the changes to the pgpool.conf file and then run pcp_reload_config to load the changes.

Edit **/etc/pgpool-II/pgpool.conf** and add the following to the file.


    # --- User routing

    user_redirect_preference_list = 'postgres:primary,reader:standby'


Save the changes and reload the pgpool config

    pcp_reload_config -h localhost -U pcpadmin

If all goes well, you will see the following ..

    pcp_reload_config -- Command Successful




