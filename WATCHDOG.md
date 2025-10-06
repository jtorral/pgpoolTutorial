
# The ultimate guide to Pgpool with Watchdog

### Presented by Jorge Torralba

  

All references to Pgpool in this document refer to Pgpool-II version 4.3 and higher. For the Docker containers used in this tutorial, we use version 4.6.3

  

## Note on Docker deployment

This guide assumes you're exploring how to set up PostgreSQL with Pgpool and Watchdog in a containerized environment. To streamline the learning process, much of the Docker configuration has been pre defined to enable rapid deployment. However, because this is intended as an educational resource, I will provide a comprehensive breakdown of the deployment steps toward the end of the documentation. This will give you clear insight into how the environment was constructed and allow you to replicate or adapt it confidently.

  

## What is Pgpool

  

Pgpool is a middleware proxy that sits between the client application and one or more Postgres database servers.

Its primary purpose is to enhance the performance, high availability, and scalability of a Postgres multi server environment by offering the following.

-   Connection pooling
    
-   Load balancing
    
-   High Availability and Failover
    

## What is Watchdog

  

Pgpool's Watchdog is a high availability feature that prevents the Pgpool middleware itself from becoming a single point of failure in a database cluster.

In simple terms, Watchdog manages a cluster of multiple Pgpool instances in an Active/Standby configuration.

  

-   The Watchdog process on each Pgpool node constantly monitors the health and status of the other Pgpool nodes in the cluster.
    
-   If the current Active Pgpool node fails, the Watchdog on the remaining Standby nodes will coordinate to elect a new Active node.
    
-   To make the switch transparent to the client applications, the Watchdog manages a shared Virtual IP Address. When a Standby node is promoted to Active, it automatically takes ownership of the VIP, ensuring that applications continue to connect to the same address without interruption.
    
-   By requiring a quorum it ensures that only one Pgpool instance can be Active at any time, preventing a damaging "split brain" scenario where multiple nodes mistakenly believe they are the current leader.
    

  

## How the Watchdog delegateIP (VIP) works

  

The Watchdog Virtual IP is a floating IP address that always points to the active Pgpool node in the cluster. It’s the single, stable entry point for clients no matter which node is currently in charge.

### A high level overview of the process.

1.  Multiple Pgpool nodes run in the cluster, each with Watchdog enabled.
    
2.  One Watchdog node is elected as the active coordinator.
    
3.  That node claims the VIP using ifconfig or ip addr add, binding it to its network interface.
    
4.  If the active node fails, Watchdog triggers a failover
    
5.  A surviving node is elected as the new coordinator.
    
6.  It then takes over the VIP, ensuring clients reconnect without needing a different connection string.
    

  

## What is needed for a failover to occur.

  

Watchdog relies on a quorum before a decision is made to failover.

A quorum is the minimum number of nodes that must be available and agreed for the cluster to consider itself operational. This is essential for making failover and leadership decisions.

  
  

## Terminology Clarification: Servers vs. Nodes

In both documentation and informal discussion, the terms servers and nodes are often used interchangeably. To improve clarity, we’ll adopt more precise definitions throughout this guide:

-   Postgres nodes refer to containers or servers running both Postgres and Pgpool.
    
-   Watchdog nodes refer specifically to containers or servers where the Pgpool Watchdog process is actively running.
    

Therefore, any reference to Watchdog nodes should be understood as the designated hosts responsible for Watchdog coordination and quorum participation.

  

## Determining what we need for our environment

  

The objective is to set up a 3 Postgres node environment that should be available even if 2 of the 3 Postgres nodes fail.

When designing a resilient environment, it’s important to understand two key questions:

1.  How many Postgres nodes (primary/standby) are in the cluster?
    
2.  What is the minimum number of nodes required to keep the cluster operational if others fail?
    

There is also the key role of Witness nodes which sometimes gets overlooked.

Witness nodes are standalone servers used solely for quorum and arbitration, not running Postgres or Pgpool directly. However, in this guide, we will be using Pgpool on the Witness servers to manage Watchdog. It’s just easier since it is already part of the Docker deploy.

Now that we are aware of the requirements based on the objectives noted above, we can move forward.

The variables for our calculations.

-   N = 3 ( Postgres nodes )
    
-   F = 2 ( Failed that can be lost out of N )
    

### The simple core rule to remember and understand

  

Pgpool Watchdog uses a quorum based voting system to determine whether the cluster can continue operating after a failure. The key rule is:

The number of healthy Watchdog nodes must be greater than the number of failed Watchdog nodes.

This ensures that any decisions such as failover or continued operation are made by a majority of active participants.

  

**For example,**

In this tutorial our cluster has 5 Watchdog nodes. How we determined 5 is detailed a little further shown below. If 2 of the Watchdog nodes fail (F), we still have 3 healthy Watchdog nodes which is a majority, so the cluster remains operational.

However, if we only had 3 Watchdog nodes and 2 failed, the remaining single node does not have quorum. Even if the Postgres backends are healthy, Pgpool will not proceed with failover or recovery actions because the Watchdog consensus is broken.

  

### Avoid confusion. Know the differences.

It’s important not to confuse Postgres node health with Watchdog node votes. Only Watchdog participants contribute to quorum decisions. Generally speaking, Postgres nodes that do not run Watchdog do not vote. But since our deployment runs Watchdog on the Postgres nodes, they are part of the entire Watchdog cluster.

  

With that out of the way, lets move on again.

### Our formula variables explained.

-   ***N*** = The number of Pgpool nodes that also run Postgres (Postgres + Watchdog nodes)
    
-   ***F*** = The number of Pgpool nodes (with Postgres) that can fail without halting the cluster ( Postgres + Watchdog nodes )
    
-   ***W*** = Number of additional Watchdog only witness nodes required to maintain quorum. (Watchdog nodes only)
    
-   ***L*** = Number of Watchdog nodes expected to remain alive after failure. (Remaining Watchdog nodes)
    
-   ***T*** = Total number of Watchdog nodes in the cluster (All Watchdog nodes)
    
-   ***Q*** = Minimum number of healthy Watchdog nodes required to maintain quorum. (Watchdog votes needed)
    

  
  
  

### Calculations

-   ***N*** = 3
    
-   ***F*** = 2
    

  

### Witness nodes (W).

-   ***W*** = ceil ( ( 2 x F + 1) - N )
    
-   ***W*** = ceil ( ( 2 x 2 + 1) - 3 )
    
-   ​***W*** = 2
    

  

### Total nodes (T).

-   ***T*** = ( W+N )
    
-   ***T*** = ( 2 + 3 )
    
-   ***T*** = 5
    

  

### Quorum (Q).

-   ***Q*** = floor ( T/2 ) + 1
    
-   ***Q*** = floor ( 5 / 2 ) + 1
    
-   ***Q***= 3
    

  

### Remaining live nodes (L).

-   ***L*** = ( N - F ) + W
    
-   ***L*** = ( 3 - 2 ) + 2
    
-   ***L*** = 3
    

### Interpretation

-   Since **L = 3** and **Q = 3**, the cluster can maintain operation when 2 of the 3 Postgres nodes are down, thanks to the 2 witness nodes.
    
-   If **L < Q** the cluster would halt to avoid data corruption such as a split brain.
    

  

### Make sure you have a clear understanding of the following

-   If you start with an odd number of Postgres servers (**3**) you can use an even number of Witness servers including 0 to keep it the total odd.
    
-   If you start with an even number of Postgres servers (**4**) AND T equates to an even number, you MUST add another Witness to make T odd, ensuring that a majority can be established and preventing ties.
    

  

### To recap the above

-   To tolerate the failure of 2 nodes in a 3 node environment, we need 2 witness nodes.
    
-   The total cluster would consist of 5 nodes. 3 Postgres nodes and 2 witness nodes.
    
-   For even numbered Postgres nodes, adding an extra witness node ensures the total **T** becomes odd, preventing tie votes and split brain.
    

  
  

## Building our environment

  

This tutorial uses Docker containers as the foundation for building out the server environment. The Git repository linked below includes all necessary Dockerfiles and configuration assets to generate the container images required for Postgres and Pgpool. While many core settings are preconfigured to streamline setup, not all requirements are fully addressed within the container build. Throughout the guide, I’ll highlight key differences between Docker based and non Docker deployments so you can adapt the instructions accordingly if you choose to build out standalone servers instead.

  

[https://github.com/jtorral/Pg17Rocky9Pgpool](https://github.com/jtorral/Pg17Rocky9Pgpool)

  

After you create the docker image from the Docker files in the above repo, you will need to start the containers in a custom network suitable for our deployment. Therefore we typically first create the network.

### Create the network.

Typically, Docker creates networks and adds both ipv4 and ipv6 protocols to the network. I have found that with the Docker containers and Pgpool at least in this environment, that only using ipv4 is a better choice.

The following command should create the network poolnet with ipv6 disabled and a subnet of **172.28.0.0** .

    docker network create --driver bridge --subnet 172.28.0.0/16 --gateway 172.28.0.1 --opt com.docker.network.enable_ipv6=false poolnet

  

### Create the Postgres containers

As already noted above, we will need 3 Postgres containers. We will name these pg1, pg2 and pg3. We will also assign them specific I.P. addresses relative to their name for easy identification.

  

**For pg1**

    docker run -p 6431:5432 -p 9991:9999 --env=PGPASSWORD=postgres -v pg1-pgdata:/pgdata --hostname=pg1 --network=poolnet --name=pg1 --privileged --ip 172.28.0.11 --sysctl net.ipv6.conf.all.disable_ipv6=1 -dt rocky9_pg17_pgpool

**For pg2**

    docker run -p 6432:5432 -p 9992:9999 --env=PGPASSWORD=postgres -v pg2-pgdata:/pgdata --hostname=pg2 --network=poolnet --name=pg2 --privileged --ip 172.28.0.12 --sysctl net.ipv6.conf.all.disable_ipv6=1 -dt rocky9_pg17_pgpool

**For pg3**

    docker run -p 6433:5432 -p 9993:9999 --env=PGPASSWORD=postgres -v pg3-pgdata:/pgdata --hostname=pg3 --network=poolnet --name=pg3 --privileged --ip 172.28.0.13 --sysctl net.ipv6.conf.all.disable_ipv6=1 -dt rocky9_pg17_pgpool

Note the external port mapping also has unique port numbers relative to the pg hostname. For example, the pg**1** internal postgres port is mapped to 643**1**, pg**2** is mapped to 643**2** and pg**3** is mapped to 643**3**. The same applies to the Pgpool port mapping starting with 999**x**

Having these port mappings allow us to access Postgres inside the container from our localhost without needing to exec into the containers directly.

  

### Create the Watchdog Witness node containers

Lastly we need the additional 2 Witness nodes to satisfy our quorum

  

**For node wd1**

    docker run -p 9994:9999 --env=PGPASSWORD=postgres -v wd1-pgdata:/pgdata --hostname=wd1 --network=poolnet --name=wd1 --privileged --ip 172.28.0.21 --sysctl net.ipv6.conf.all.disable_ipv6=1 -dt rocky9_pg17_pgpool

  

**For node wd2**

    docker run -p 9995:9999 --env=PGPASSWORD=postgres -v wd2-pgdata:/pgdata --hostname=wd2 --network=poolnet --name=wd2 --privileged --ip 172.28.0.22 --sysctl net.ipv6.conf.all.disable_ipv6=1 -dt rocky9_pg17_pgpool

  

### Perform on all containers

  

Since the services in this deployment run under the postgres user, and Pgpool’s Watchdog requires the ability to bring the virtual IP up and down from that user context, we need to grant specific OS level privileges to allow this without requiring a password.

Docker exec into each container and as root do perform the following:

  
    docker exec -it <container_name> /bin/bash

  
Edit the sudoers file:

    visudo

  

Append the following lines to the end of the file

    postgres ALL=NOPASSWD: /usr/sbin/ip  
    postgres ALL=NOPASSWD: /usr/sbin/arping

  

If you're not using the Docker files provided in this tutorial, verify the correct paths for ip and arping on your system. These may vary depending on your distribution or container base image.

  
  

These changes ensure that the postgres user can manage the VIP interface without being prompted for a password, a requirement for seamless Watchdog failover and recovery operations.

  
  

### Prepare the Postgres server

  

To streamline the setup, the Docker containers come pre configured with the Postgres database server and SSH enabled. This configuration provides the postgres user with passwordless connections between all containers, which is necessary for their communication and a requirement for some of the features we will use from Pgpool in our tutorial.

Let me say it now. It's important to acknowledge that one of the most complex and often frustrating aspects of working with Pgpool is its authentication model. The interactions between users, clients, servers, administrative tools, and Pgpool itself can be difficult to untangle. Until you fully understand the underlying mechanisms, the process can feel chaotic and inconsistent. Pgpool does not follow a uniform approach to authentication, which makes troubleshooting and configuration particularly challenging for newcomers and seasoned operators alike.

With that said, we will be using MD5 authentication in this guide. Some steps are needed to prepare Postgres for this and they are outlined below.

  

#### Log onto pg1

  

    docker exec -it pg1 /bin/bash

su to postgres

    su - postgres

  

Side note. In this setup, we use a separate file named pg_custom.conf to manage Postgres specific customizations, rather than modifying postgresql.conf directly. This file is included at the end of postgresql.conf, allowing us to isolate and track changes more easily. This approach simplifies configuration management and makes it easier to maintain consistency across environments.

  

Modify the postgresql.conf file.

    cd $PGDATA

  

The above cd command should put you in the /pgdata/17/data directory of the container.

Make the modifications to the file pg_custom.conf

    vi pg_custom.conf

  

Append the following line to the end of the file then save the changes

    password_encryption = md5

Modify the pg_hba.conf file

Find any references to **scram-sha-256** and change them to **md5**. Afterwards, the file should look like the one shown below.

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
 

We can now start Postgres and perform some minor necessary steps.

    pg_ctl start

  

#### Create accounts needed by pgpool

  

Start a psql session

    [postgres@pg1 ~]$ psql  
    psql (17.6)  
    Type "help" for help.  
      
    postgres=#

  

Create the following roles.

    create  role pgpool with login password 'pgpool';  
    create  role replicator with  replication login password 'replicator';

  

Reset the postgres password so it is encrypted in md5. Originally, the containers were using scram-sha-256 encryption.

    alter  role postgres with  password 'postgres';  

#### Create the needed extension

  

Pgpool needs the extension pgpool_recovery created in the template1 database in order to run certain commands against the database.

  

Connect to the template1 database and create the extension.

    \c template1

  

Create the extension in the template1 database

    create extension pgpool_recovery;

  
  
  
  
  
  

## Prepare Pgpool and Watchdg.

  

As previously discussed, we'll now shift our focus to the setting up the Pgpool side of things on both the Postgres and dedicated Watchdog Witness servers.

The next phase involves configuring each server to enable Pgpool’s core functionality, including its command line management interface known as PCP tools. These tools allow administrators to remotely control and monitor Pgpool including node state management, health checks, recovery triggers, and inspection of internal status.

To streamline setup, you can configure one node and replicate the configuration files to the others. Starting with Pgpool version 4.3, configuration files such as pgpool.conf, pcp.conf, and pool_hba.conf can be shared across nodes without requiring node specific customization, provided your deployment uses consistent hostnames and network settings.

  
  

### A rant about authentication in Pgpool

I really need to get this out of the way. I am simply the messenger.

As I mentioned earlier, authentication in Pgpool is notoriously complex and frustrating. It's really more a reflection of its evolution than a unified design. Keep in mind, Pgpool has been around a very long time and has vastly improved and become a stable HA solution for many enterprise level deployments. Until you understand these nuances, it can feel like deciphering a different secret handshake for every subsystem.

Take the pcpadmin user, for example. To authenticate with Pgpool’s PCP interface, their password must be stored as an MD5 hash in pcp.conf. However, when you run a PCP command like pcp_node_info, the client utility expects a plain-text password from the .pcppass file. These two components only work together if the MD5 hash in pcp.conf corresponds to the plain text password in .pcppass.

Then there’s the recovery_user defined in pgpool.conf, responsible for connecting to the backend Postgres servers during recovery. You might expect it to follow the same MD5 convention but it doesn’t. The recovery_password must be either plain text or AES-256 encrypted. So while PCP users rely on MD5, recovery users operate under a completely different protocol.

The inconsistency continues. When pg_basebackup is invoked during recovery, it uses the password from recovery_password (or a .pgpass file) to authenticate with Postgres. This password must be plain text, as Postgres does not accept hashed credentials for replication connections.

In short, Pgpool’s authentication model juggles MD5 hashes, plain text passwords, and AES encryption, depending on the context. There’s no single correct format, only the correct format for the specific action you’re performing. This makes it really difficult to debug or identify simple authentication errors. Especially for those just learning how to use Pgpool.

The silver lining? Once everything is properly configured and aligned, Pgpool authentication becomes stable and predictable. But getting there requires careful attention to detail and a deep understanding of how each component interacts.

  

### PCP Tools

An essential part of this configuration is setting up the PCP (Pgpool Control Protocol) tools. These are a set of command line utilities that allow us to interact with and manage the Pgpool instance directly. Rather than editing configuration files and restarting the service, For example, we can use PCP tools to perform administrative tasks such as:

-   Checking the status of each Postgres backend node.
    
-   Attaching or detaching a node from the cluster.
    
-   Promoting a replica to become the new primary server.
    
-   Displaying health check statistics.
    
-   Reloading the Pgpool configuration without stopping the service.
    

  

For your reference, here is a list of what you can accomplish with the PCP  suite of command line tools.

  

-   **pcp_node_count**: Returns the total number of backend nodes currently managed by Pgpool. Useful for confirming cluster size.
    
-   **pcp_node_info**: Provides detailed information about a specific node, including its status (up/down), role (primary/standby), and connection state.
    
-   **pcp_detach_node**: Temporarily removes a node from Pgpool’s active backend list. This is typically used when a node is unhealthy or undergoing maintenance.
    
-   **pcp_attach_node**: Reattaches a previously detached node back into Pgpool’s backend pool. Often used after recovery or manual intervention.
    
-   **pcp_recovery_node**: Triggers the recovery process for a failed node. This invokes the configured recovery scripts to resynchronize the node with the primary.
    
-   **pcp_watchdog_info**: Displays the current status of the Watchdog subsystem, including which node holds the delegate IP and the state of each Watchdog participant.
    
-   **pcp_pool_status**: Dumps the current Pgpool configuration parameters and runtime state. Useful for debugging and auditing.
    
-   **pcp_proc_count**: Shows how many child processes Pgpool has spawned to handle client connections. This helps monitor load and concurrency.
    
-   **pcp_proc_info**: Provides detailed information about a specific Pgpool child process, including its connection details and assigned backend.
    
-   **pcp_health_check_stats**: Displays health check statistics for each backend node, including success/failure counts and timestamps. Crucial for diagnosing flapping nodes or connectivity issues.
    

When you authenticate with PCP tools, the authentication method is completely separate from the regular Postgres user accounts.

PCP uses its own, independent authentication mechanism defined in Pgpool's configuration file, typically **pcp.conf**. Instead of relying on usernames and passwords stored within the Postgres database, you create a dedicated user and a hashed password specifically for administering the Pgpool instance. This means that a user account used to manage Pgpool, does not have to be a valid user within the Postgres database itself.

  

### Setting up pcp

  

Open up another terminal session and log on to the docker container. ( **wd1**). Lets just use wd1 as our source moving forward unless noted otherwise.

    docker exec -it  wd1 /bin/bash

sudo to postgres

    su - postgres

cd to the pgpool directory

    cd /etc/pgpool-II

The pcp.conf file has the credentials for the user stored in the following format

    <username>:<md5 password hash>

For this tutorial the username will be **pcpadmin** and the password will also be **pcpadmin**.

To generate and md5 hash, from the command line run the following

    pg_md5 pcpadmin

This will generate the following

    3090be164580c74a25efe7abdf542eb2

Copy the hash generated above.

    vi pcp.conf

Clear out the existing content and append the following line to the file.

    pcpadmin:3090be164580c74a25efe7abdf542eb2

Save the file and make sure the permissions are set to 600.

    chmod 600 pcp.conf

  

### The .pcppass file

The .pcppass file is an optional, client side file used to store the credentials for connecting to the Pgpool instance using the PCP tools. Instead of typing the password every time you run a command, the tool can read it from this file.

  

The file is located **in the user's home directory** and must have strict permissions of 0600 to protect the credentials. Since we are going to be using the postgres account to run most of the Pgpool commands in our instance, let's set up the **.pcppass** file for the user postgres.

At this point we should still be logged in as postgres after we did the **su - postgres** earlier.

  

The format is simple: `hostname:port:user:password`

  

The password will be in plain text which is why the file permissions need to be 600.

    cd \~

Create the .pcpass file

    vi .pcppass

Add the following line to the file. This ensures we have the credentials saved for all nodes in our cluster.

    *:9898:pcpadmin:pcpadmin

  

We are using a wildcard (*) for the host field of the file. This means we can run this from any node without having to declare each node specifically in the .pcppass file.

  

Save your changes and modify the permissions to 600

    chmod 600 .pcppass

  

By default, pcp listens on port 9898. It is also defined in our pgpool.conf file.

Remember, the credentials in this file are for logging into and managing Pgpool itself, not for connecting to the database.

### The pool_passwd file

  

The pgpool_passwd file is a critical configuration file used by Pgpool to securely store the encrypted passwords for connecting to the backend Postgres database servers. Pgpool uses these credentials to authenticate itself to the primary and replica nodes, which is necessary for core features like replication and failover to function correctly. This file ensures that Pgpool can securely manage its connections to the databases without storing plaintext passwords.

  

### Generate the pool_passwd file

  

If you're using MD5 based authentication as described in this documentation and need to populate the pool_passwd file with multiple users, you can streamline the process by running the following query directly against your Postgres database:

    select usename || ':' || passwd from pg_shadow;

  

This will output each username along with its MD5-hashed password in the format expected by pool_passwd. You can copy and paste the results directly into the file, saving you from manually generating and formatting each entry.

**Note**: This assumes your Postgres users already have MD5 passwords set. If you're not using MD5 or your environment uses a different password scheme, this method will not apply.

If you prefer to manually generate entries for the pool_passwd file, you can use the pg_md5 utility provided by Pgpool. This tool hashes passwords and formats them correctly for Pgpool authentication.

    cd /etc/pgpool-II/

Run the following command for each user you wish to add:

    # For the pgpool user  
    pg_md5 -m -u pgpool pgpool  
      
    # For the replicator user  
    pg_md5 -m -u replicator replicator  
      
    # For the postgres user  
    pg_md5 -m -u postgres postgres

  

Each command appends a new entry to the pool_passwd file in the format **username:md5hash**. If you have additional users, simply repeat the command with their respective usernames and passwords.

After running the above commands, your pool_passwd file will contain entries like:

    pgpool:md5f24aeb1c3b7d05d7eaf2cd648c307092  
    replicator:md55577127b7ffb431f05a1dcd318438d11  
    postgres:md53175bce1d3201d16594cebf9d7eb3f9d

  

### The .pgpass file

  

The .pgpass file is a file used by Postgres client applications, including tools like psql and pg_basebackup to store passwords for connecting to a Postgres database. It works with Pgpool by providing the password needed to connect to the backend database nodes.

The .pgpass file is designed to automate the password entry process. When a Postgres client application tries to connect to a server, it checks for a .pgpass file in the user's home directory. It reads the file line by line, looking for an entry that matches the connection details:

-   Hostname
    
-   Port
    
-   Database name
    
-   Username
    
-   Password
    

If a line matches all four criteria, the client automatically uses the plain text password from that line and sends it to the server for authentication. This allows you to run scripts and commands without being manually prompted for a password.

**When it comes to Pgpool, the .pgpass file is most crucial for the recovery process**. When pcp_recovery_node is executed, Pgpool runs a recovery_1st_stage_command (usually pg_basebackup) on a backend node. This command needs a password to connect to the primary Postgres database, and it will look for that password in a .pgpass file located on the backend server, in the home directory of the user running the command.

Remember those authentication inconsistencies I mentioned earlier? This is where they come into play. For example, a **replication_user** defined in **pgpool.conf cannot use md5 for password**. So, if your database is set to use md5 as in this tutorial, and you have the user defined in .pgpass, you will get an error. The only workarounds are to use AES encryption **or put the password in plain text in the config file.**

  

### Create the .pgpass file in the postgres home directory.

To enable seamless authentication for PostgreSQL utilities and Pgpool recovery operations, create a .pgpass file in the home directory of the postgres user.

  

Ensure you are logged in as user postgres

    cd \~

Create or edit the .pgpass file

    vi .pgpass

Add the following entries.

    *:*:*:postgres:postgres  
    *:*:*:replicator:replicator  
    *:*:*:pgpool:pgpool

**Important. Make sure there are no spaces when you copy and paste.**

Save the changes and set the proper file permissions

    chmod 600 .pgpass

  

**Note the following regarding .pgpass**

Since the same credentials are used across all nodes in this deployment, you can simplify the .pgpass entries by using wildcards (*) for the host, port, and database fields. However, wildcards cannot be used for the username or password, those must be explicitly defined.

## Propagate the files to the other nodes.

  

Once the necessary configuration files have been created, you’ll need to distribute them across all nodes in the cluster. Since the Docker containers are preconfigured with SSH keys, you can use scp to securely copy the files to their respective destinations.

Assuming the edits were made on node wd1, execute the following commands:
  

### Copy .pgpass to all nodes

  

    scp /var/lib/pgsql/.pgpass  pg1:/var/lib/pgsql/  
    scp /var/lib/pgsql/.pgpass  pg2:/var/lib/pgsql/  
    scp /var/lib/pgsql/.pgpass  pg3:/var/lib/pgsql/  
    scp /var/lib/pgsql/.pgpass  wd2:/var/lib/pgsql/

  

### Copy .pcppass to all nodes

  

    scp /var/lib/pgsql/.pcppass  pg1:/var/lib/pgsql/  
    scp /var/lib/pgsql/.pcppass  pg2:/var/lib/pgsql/  
    scp /var/lib/pgsql/.pcppass  pg3:/var/lib/pgsql/  
    scp /var/lib/pgsql/.pcppass  wd2:/var/lib/pgsql/

  
  

### Copy pcp.conf to all nodes

  

    scp /etc/pgpool-II/pcp.conf pg1:/etc/pgpool-II/  
    scp /etc/pgpool-II/pcp.conf pg2:/etc/pgpool-II/  
    scp /etc/pgpool-II/pcp.conf pg3:/etc/pgpool-II/  
    scp /etc/pgpool-II/pcp.conf wd2:/etc/pgpool-II/

  

### Copy pool_passwd to all nodes

  

    scp /etc/pgpool-II/pool_passwd pg1:/etc/pgpool-II/  
    scp /etc/pgpool-II/pool_passwd pg2:/etc/pgpool-II/  
    scp /etc/pgpool-II/pool_passwd pg3:/etc/pgpool-II/  
    scp /etc/pgpool-II/pool_passwd wd2:/etc/pgpool-II/

  

### Recovery script files

For this Docker based deployment, the recovery scripts (recovery_1st_stage and pgpool_remote_start) have already been pre edited and placed in the appropriate locations. These scripts are tailored to work seamlessly within the containerized environment, so no immediate changes are required to get started.

However, if you're customizing your setup or working outside of this Docker context, we’ll cover how to copy the original scripts from the sample_scripts directory and modify them to suit your environment. That section appears later in this documentation and includes guidance on adapting paths, permissions, and command logic as needed.

  

### Copy recovery scripts to pg1 only

    scp /etc/pgpool-II/recovery_1st_stage pg1:/pgdata/17/data
    scp /etc/pgpool-II/pgpool_remote_start pg1:/pgdata/17/data

  

**Note:** We copy the recovery scripts only to pg1 because it serves as the initial primary database node during cluster startup.

  

## Central Configuration

  

As noted earlier, Pgpool now supports a unified configuration model, allowing a single pgpool.conf file to be shared across all nodes in the cluster. This simplifies deployment and ensures consistency.

The main configuration file, pgpool.conf, is located at:

    /etc/pgpool-II/pgpool.conf

  

This file contains all the core settings required to initialize and operate the Pgpool environment. For the purposes of this guide, using this shared configuration should be sufficient to get our deployment for this guide up and running.

Later in the guide, we’ll take a closer look at some of the less obvious parameters to help you better understand their roles and fine tune your setup with confidence.

  

## The pgpool.conf file

Still logged in as user postgres on wd1, edit the pgpool.conf file.

    cd /etc/pgpool-II/

Edit the file

    vi pgpool.conf

Replace it’s current content with the following:

    pool_passwd                    = '/etc/pgpool-II/pool_passwd'
    auth_type                      = 'md5'
    enable_pool_hba                = off
    authentication_timeout         = 60
    
    #--- Connection Settings ---
    
    listen_addresses                = '*'
    port                            = 9999
    
    #--- pcp connection Settings ---
    
    pcp_socket_dir                  = '/tmp'
    pcp_listen_addresses            = '*'
    pcp_port                        = 9898
    
    #--- Replication stuff
    
    sr_check_period         = 10
    sr_check_user           = 'replicator'
    sr_check_password       = ''
    sr_check_database       = 'postgres'
    
    
    
    #--- Load Balancing and Query Routing ---
    
    load_balance_mode               = on
    reset_query_on_pool_release     = on
    replicate_on_reset              = on
    
    #--- Backend Server Configuration ---
    
    backend_hostname0               = 'pg1'
    backend_port0                   = 5432
    backend_weight0                 = 1
    backend_flag0                   = 'ALLOW_TO_FAILOVER'
    backend_data_directory0         = '/pgdata/17/data'
    backend_clustering_mode0        = 'streaming_replication'
    backend_application_name0       = 'pg1'
    
    backend_hostname1               = 'pg2'
    backend_port1                   = 5432
    backend_weight1                 = 1
    backend_flag1                   = 'ALLOW_TO_FAILOVER'
    backend_data_directory1         = '/pgdata/17/data'
    backend_clustering_mode1        = 'streaming_replication'
    backend_application_name1       = 'pg2'
    
    backend_hostname2               = 'pg3'
    backend_port2                   = 5432
    backend_weight2                 = 1
    backend_flag2                   = 'ALLOW_TO_FAILOVER'
    backend_data_directory2         = '/pgdata/17/data'
    backend_clustering_mode2        = 'streaming_replication'
    backend_application_name2       = 'pg3'
    
    #--- failover and recovery info
    
    failover_on_backend_error        = on
    failover_command                 = '/etc/pgpool-II/failover.sh %d %h %p %D %m %H %M %P %r %R %N %S'
    follow_primary_command           = '/etc/pgpool-II/follow_primary.sh %d %h %p %D %m %H %M %P %r %R'
    recovery_user                    = 'postgres'
    recovery_password                = 'postgres'
    recovery_1st_stage_command       = 'recovery_1st_stage'
    
    
    #--- Watchdog stuff ---
    
    health_check_user                = 'pgpool'
    health_check_password            = 'pgpool'
    health_check_database            = 'template1'
    
    
    use_watchdog                      = on
    delegate_ip                       = '172.28.0.100'
    if_up_cmd                         = '/usr/bin/sudo /usr/sbin/ip addr add 172.28.0.100/24 dev eth0'
    if_down_cmd                       = '/usr/bin/sudo /usr/sbin/ip addr del 172.28.0.100/24 dev eth0'
    arping_cmd                        = '/usr/bin/sudo /usr/sbin/arping -U 172.28.0.100 -w 1 -I eth0'
    
    wd_interval                      = 20 # How often to run a check in seconds
    
    wd_lifecheck_method              = 'query'
    wd_lifecheck_query               = 'SELECT 1'
    wd_lifecheck_dbname              = 'template1'
    wd_lifecheck_user                = 'pgpool'
    wd_lifecheck_password            = 'pgpool'
    wd_life_point                    = 5   # number of times to retry a failed life check
    
    hostname0                        = 'wd1'
    wd_port0                         = 9000
    pgpool_port0                     = 9999
    
    hostname1                        = 'wd2'
    wd_port1                         = 9000
    pgpool_port1                     = 9999
    
    hostname2                        = 'pg1'
    wd_port2                         = 9000
    pgpool_port2                     = 9999
    
    hostname3                        = 'pg2'
    wd_port3                         = 9000
    pgpool_port3                     = 9999
    
    hostname4                        = 'pg3'
    wd_port4                         = 9000
    pgpool_port4                     = 9999
    
    #--- disable cache
    
    statement_cache_mode             = off
    query_cache_mode                 = off
    
    #--- logging info
    
    logging_collector                = on
    log_min_message                  = DEBUG1
    log_destination                  = 'stderr'
    log_line_prefix                  = '%m: %a pid %p: '
    log_directory                    = '/var/log/pgpool_log'
    log_filename                     = 'pgpool-%a.log'
    log_truncate_on_rotation         = on
    log_rotation_age                 = 1d
    log_rotation_size                = 0
    log_per_node_statement           = on
    notice_per_node_statement        = on


Save the changes

  

### Copy pgpool.conf to all nodes

    scp /etc/pgpool-II/pgpool.conf pg1:/etc/pgpool-II/  
    scp /etc/pgpool-II/pgpool.conf pg2:/etc/pgpool-II/  
    scp /etc/pgpool-II/pgpool.conf pg3:/etc/pgpool-II/  
    scp /etc/pgpool-II/pgpool.conf wd2:/etc/pgpool-II/



### Assign pgpool_node_id to each node.

**Important**. Do not forget to do this on each node.

Each server needs the file /etc/pgpool-II/pgpool_node_id. The file will simply have a number matching the defined hostname in the pgpool.conf file.

  

For example, we can see that in our pgpool.conf file the following is set. hostname3 = ‘pg2’

This means that on the actual server pg2 the file /etc/pgpool-II/pgpool_node_id must simply contain 1 line with the number 3.

And based on the configuration shown above, The file /etc/pgpool-II/pgpool_node_id on server wd1 would contain 1 line with the number 0.

From **wd1** you could simply run the following grep to identify the hostnames in the pgpool.conf

  

    cat pgpool.conf | grep hostname | grep -v backend  
      
    hostname0 = 'wd1'  
    hostname1 = 'wd2'  
    hostname2 = 'pg1'  
    hostname3 = 'pg2'  
    hostname4 = 'pg3'


Using the above output, we can now easily assign the node id’s for each node.

    ssh wd1 "echo 0 > /etc/pgpool-II/pgpool_node_id"
    ssh wd2 "echo 1 > /etc/pgpool-II/pgpool_node_id"
    ssh pg1 "echo 2 > /etc/pgpool-II/pgpool_node_id"
    ssh pg2 "echo 3 > /etc/pgpool-II/pgpool_node_id"
    ssh pg3 "echo 4 > /etc/pgpool-II/pgpool_node_id"


## Ready to startup the service

  

On each node as user postgres run the following

    pgpool -f /etc/pgpool-II/pgpool.conf

  

This will start Pgpool on each node.

  

After you start Pgpool on all nodes, from any node run the command

    pcp_node_info -U pcpadmin -h localhost | column -t

  

If you get an output that is not an error, we are good to go.

If you get an output similar to the following:

    ERROR: connection to host "localhost" failed with error "Connection refused"

  
Check the Postgres log files on pg1

If you see authentication message like the following

    2025-10-06 05:01:22.795 UTC [172.28.0.11(52620)] [2723]: [1-1] user=replicator,db=postgres,host=  
    172.28.0.11 FATAL: password authentication failed for user "replicator"

  

Chances are you have spaces at the end of each line in your .pgpass file or pool_passwd file. If you do, remove any extra spaces, copy the files to all the nodes and restart Pgpool.

You can restart Pgpool by stopping it on all nodes.

    pgpool -f /etc/pgpool-II/pgpool.conf stop

  
And then restarting it on all nodes afterwards.

    pgpool -f /etc/pgpool-II/pgpool.conf

  

### Start the standby nodes

  

At this point we should only have pg1 up and running.
  

To start the other nodes and sync them with pg1, run the following

**Starting pg2**

    pcp_recovery_node -U pcpadmin -h localhost -n 1

  

**Starting pg3**

When pg2 is up and running, start pg3

    pcp_recovery_node -U pcpadmin -h localhost -n 2

  

### View the current status of the database cluster

  

    [postgres@wd1 ~]$ pcp_node_info -U pcpadmin -h localhost | column -t  
    
    pg1 5432 2 0.333333 up unknown primary unknown 0 none  none 2025-10-06 05:09:52  
    pg2 5432 2 0.333333 up unknown standby unknown 0 none  none 2025-10-06 05:09:52  
    pg3 5432 2 0.333333 up unknown standby unknown 0 none  none 2025-10-06 05:09:52

  

### View the current status of the Watchdog nodes

  
  

pcp_watchdog_info -U pcpadmin -h localhost | column -t  

    pcp_watchdog_info -U pcpadmin -h localhost | column -t
    
    5         5      NO   pg2:9999  Linux  pg2   pg2               
    wd1:9999  Linux  wd1  wd1       9999   9000  7    STANDBY  0  MEMBER
    wd2:9999  Linux  wd2  wd2       9999   9000  7    STANDBY  0  MEMBER
    pg1:9999  Linux  pg1  pg1       9999   9000  7    STANDBY  0  MEMBER
    pg2:9999  Linux  pg2  pg2       9999   9000  4    LEADER   0  MEMBER
    pg3:9999  Linux  pg3  pg3       9999   9000  7    STANDBY  0  MEMBER




### pgpool.conf explanation and REQUIRED action.

As mentioned above, we will break down the sections of the config file and explain what they represent. To simplify the details, we will focus on what does not appear to be obvious.

  

### Backend server configuration explained


    backend_hostname0               = 'pg1'
    backend_port0                   = 5432
    backend_weight0                 = 1
    backend_flag0                   = 'ALLOW_TO_FAILOVER'
    backend_data_directory0         = '/pgdata/17/data'
    backend_clustering_mode0        = 'streaming_replication'
    backend_application_name0       = 'pg1'
    
    backend_hostname1               = 'pg2'
    backend_port1                   = 5432
    backend_weight1                 = 1
    backend_flag1                   = 'ALLOW_TO_FAILOVER'
    backend_data_directory1         = '/pgdata/17/data'
    backend_clustering_mode1        = 'streaming_replication'
    backend_application_name1       = 'pg2'
    
    backend_hostname2               = 'pg3'
    backend_port2                   = 5432
    backend_weight2                 = 1
    backend_flag2                   = 'ALLOW_TO_FAILOVER'
    backend_data_directory2         = '/pgdata/17/data'
    backend_clustering_mode2        = 'streaming_replication'
    backend_application_name2       = 'pg3'

  

The backend server section is for defining the actual Postgres servers in the cluster. It has nothing to do with the Watchdog nodes. We will talk about that later. Our deployment will have 3 Postgres database servers, pg1, pg2 and pg3. Each of these database servers is assigned a backend inside the pgpool.conf starting with 0.

As you can see, pg1 is associated with backend 0, pg2 with backend 1 and pg3 with backend 2.

The variables are pretty obvious. They represent the hostname, port, data directory for the database server along with a few other parameters.

**For example,**

backend_hostname1 is for defining the name or ip address of the pg2 database server. backend_data_directory1 is the data directory for the pg2 server, backend_clustering_mode1 is stating streaming replication is the replication method used. Backend_weight1 is the weight the server will receive in a load balance mode. For example, if all 3 servers have a weight of 1, we should see the queries spread across each server equally. Each would get about 33% of the load.

For more details, refer to the pgpool documentation.

[https://www.pgpool.net/docs/latest/en/html/index.html](https://www.pgpool.net/docs/latest/en/html/index.html)

### Watchdog configuration settings explained

    health_check_user                = 'pgpool'
    health_check_password            = 'pgpool'
    health_check_database            = 'template1'
    
    use_watchdog                      = on
    delegate_ip                       = '172.28.0.100'
    if_up_cmd                         = '/usr/bin/sudo /usr/sbin/ip addr add 172.28.0.100/24 dev eth0'
    if_down_cmd                       = '/usr/bin/sudo /usr/sbin/ip addr del 172.28.0.100/24 dev eth0'
    arping_cmd                        = '/usr/bin/sudo /usr/sbin/arping -U 172.28.0.100 -w 1 -I eth0'
    
    wd_interval                      = 20 # How often to run a check in seconds
    
    wd_lifecheck_method              = 'query'
    wd_lifecheck_query               = 'SELECT 1'
    wd_lifecheck_dbname              = 'template1'
    wd_lifecheck_user                = 'pgpool'
    wd_lifecheck_password            = 'pgpool'
    wd_life_point                    = 5   # number of times to retry a failed life check
    
    hostname0                        = 'wd1'
    wd_port0                         = 9000
    pgpool_port0                     = 9999
    
    hostname1                        = 'wd2'
    wd_port1                         = 9000
    pgpool_port1                     = 9999
    
    hostname2                        = 'pg1'
    wd_port2                         = 9000
    pgpool_port2                     = 9999
    
    hostname3                        = 'pg2'
    wd_port3                         = 9000
    pgpool_port3                     = 9999
    
    hostname4                        = 'pg3'
    wd_port4                         = 9000
    pgpool_port4                     = 9999

  

Let's look at the configuration settings for Watchdog. We will jump around a little here and not necessarily follow them in the order you see them.

  

#### The watchdog node declarations.

  

-   hostnameX
    
-   wd_portX
    
-   pgpool_portX
    

  

These are completely different from the backend server definitions used for the Postgres server. These reference only the nodes running Watchdog. Which in our case will be 5 nodes all together. The 3 Postgres nodes (**pg1, pg2 and pg3**) and the 2 Witness servers ( **wd1 and wd1** ).  **X** above represents a unique id given to each node. In our case, 5 servers get id 0 through 4.


### The delegate IP section explained

As noted earlier, watchdog will manage a virtual ip which will float around the different Watchdog nodes. This is handled during failover and startup of the cluster and based on health checks and failures. This is the IP that clients will connect to.

  

We assign a free I.P on the subnet to Watchdog. In our case, we have chosen the ip of 172.28.0.100 as the VIP.

    delegate_ip = '172.28.0.100'


Additionally, in the pgpool.conf file make sure you specify a complete path for sudo and the ip commands.

    if_up_cmd = '/usr/bin/sudo  /usr/sbin/ip addr add 172.28.0.100/24 dev eth0'  
    if_down_cmd = '/usr/bin/sudo  /usr/sbin/ip addr del 172.28.0.100/24 dev eth0'  
    arping_cmd = '/usr/bin/sudo  /usr/sbin/arping -U 172.28.0.100 -w 1 -I eth0'


