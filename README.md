
# Pgpool step by step setup guide


This document provides a detailed training guide for setting up **pgpool-II** to achieve both **load balancing** and **high availability with failover**. 

The instructions are designed to be followed step by step using a pre configured environment of custom **Docker containers**.  This approach ensures a consistent and reproducible setup, making it an excellent resource for a training session or self-paced learning.

The provided files will also include instructions on how to download the necessary Docker files so you can create the necessary image and then create the containers for the environment.

Additionally, there is a file for creating the environment on bare metal server without using Docker.

Due to crucial differences in the authentication configurations, this guide has been divided into two separate, but structurally similar, documents. It is vital that you choose the correct file to avoid configuration errors.

**[pgpool-md5.md](https://github.com/jtorral/pgpoolTutorial/blob/main/pgpool-md5.md)**:  This file is intended for users who are configuring their system with the older, but still common, **MD5 authentication** method. It details the specific steps required to ensure proper password and user authentication between pgpool and the underlying Postgres servers using MD5.
    
**[pgpool-scram.md](https://github.com/jtorral/pgpoolTutorial/blob/main/pgpool-scram.md)**: This file is for those using the more modern and secure **SCRAM-SHA-256 authentication**. This document outlines the unique requirements and steps necessary for this authentication protocol, including specific password generation and configuration settings that differ significantly from the MD5 process.
    

**[pgpool-rocky9-md5.md](https://github.com/jtorral/pgpoolTutorial/blob/main/pgpool-rocky9-md5.md)**:  This file is intended for users who are not using the Docker approach to seeting up the environment and are installing bare metal servers for postgres and pgpool.


While the overall goal of both documents is the same, to set up **pgpool** for load balancing and failover, the details surrounding user authentication are distinct. Following the wrong guide could lead to authentication failures and an unusable setup. Leading to frustration. Therefore, please select the appropriate file based on your planned authentication method.

### PLEASE READ THIS ....

It is important to note that this guide does **not** include the most robust or ideal choices for production environments, such as:

-   **Advanced Health Checks**: The guide uses simple, default health checks. In a production setting, you would want to tune these to be more sophisticated and application aware.

- **Connection pooling**: The settings for connection pooling are at their most basic. For optimal performance, these parameters should be carefully tuned based on your application's connection patterns and workload.
    
-   **Timeouts**: The provided timeout settings are minimal. For a real world application, you would need to adjust these to your specific network and workload to avoid issues.
    
-   **Watchdog**: The document does not cover the use of **Watchdog**, a critical component for achieving high availability of pgpool itself by preventing it from being a single point of failure.
    

This guide is intended as a starting point. Once you have a working setup and a basic understanding, you should thoroughly review the pgpool II documentation to **tune the configuration** to meet the specific requirements and robustness needed for your application.



### Lastly ...

Unfortunately, I do not have an image for ARM architecture yet. That is in the works.  If you feel you have time and can modify the Docker file for a **--platform=linux/arm64 rockylinux:9** please feel free to do so and submit a PR.


### What is Pgpool ?

Pgpool is a middleware for Postgres that sits between the client and the database server. It provides several key features to enhance a Postgres database's performance, high availability, and scalability. It is not a replacement for Postgres.

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


