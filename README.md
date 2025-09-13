
# Pgpool step by step setup guide


This document provides a detailed training guide for setting up **pgpool-II** to achieve both **load balancing** and **high availability with failover**. 

The instructions are designed to be followed step by step using a pre configured environment of custom **Docker containers**.  This approach ensures a consistent and reproducible setup, making it an excellent resource for a training session or self-paced learning.

The provided files will also include instructions on how to download the necessary Docker files so you can create the necessary image and then create the containers for the environment.

Due to crucial differences in the authentication configurations, this guide has been divided into two separate, but structurally similar, documents. It is vital that you choose the correct file to avoid configuration errors.

**[pgpool-md5.md](https://github.com/jtorral/pgpoolTutorial/blob/main/pgpool-md5.md)**:  This file is intended for users who are configuring their system with the older, but still common, **MD5 authentication** method. It details the specific steps required to ensure proper password and user authentication between pgpool and the underlying Postgres servers using MD5.
    
**[pgpool-scram.md](https://github.com/jtorral/pgpoolTutorial/blob/main/pgpool-scram.md)**: This file is for those using the more modern and secure **SCRAM-SHA-256 authentication**. This document outlines the unique requirements and steps necessary for this authentication protocol, including specific password generation and configuration settings that differ significantly from the MD5 process.
    

While the overall goal of both documents is the same, to set up **pgpool** for load balancing and failover, the details surrounding user authentication are distinct. Following the wrong guide could lead to authentication failures and an unusable setup. Leading to frustration. Therefore, please select the appropriate file based on your planned authentication method.

Unfortunately, I do not have an image for ARM architecture yet. That is in the works.  If you feel you have time and can modify the Docker file for a **--platform=linux/arm64 rockylinux:9** please feel free to do so and submit a PR.


