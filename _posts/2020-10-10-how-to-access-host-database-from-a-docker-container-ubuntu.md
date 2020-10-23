---
layout: post
title: How to access host database from a docker container (Ubuntu)
---

I was trying to install self-hosted [Plausible analytics](https://docs.plausible.io/self-hosting/) for this blog, and got stuck as my current server already has a PostgreSQL database installed. I want the Plausible analytics docker container to access the host database instead of spinning up another database container, searched Stack Overflow for answers but none of them seems to work.



This post would be focusing on PostgreSQL database, and assuming you are using docker on Ubuntu, with UFW firewall switched on.



Assuming you already have installed a database server (PostgreSQL) on your host, with the port 5432 listening for connection.



The first step would be to ask PostgreSQL to listen to all address, edit the PostgreSQL config file located at **/etc/postgresql/10/main/postgresql.conf** (replace 10 with the version of your postgresql) :

`nano /etc/postgresql/10/main/postgresql.conf`



Find the **listen_addresses** in the connection and authentication block (around line 60), change its value to "*" to listen to all addresses instead of localhost, as docker container will connect using the 172.16.0.0/12 private IP range instead of 127.0.0.1.



```bash
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = '*'
```

 

Then press "ctrl+x" and save.



Next, edit the PostgreSQL Client Authentication Configuration File located at **/etc/postgresql/10/main/pg_hba.conf** .

`nano /etc/postgresql/10/main/pg_hba.conf`



Scroll to the #IPv4 local connections section, change the line below the ""#IPv4 local connections" to this :

```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             172.16.0.0/12            md5
```

 

This will allow all IP address from the 172.16.0.0/12 subnet to connect to your database (this is private IP range for local area network, which is not available for the worldwide internet to use, so don't worry about hacker for remote accessing).



Then press "ctrl+x" and save.



We have to restart the PostgreSQL service for it to use the updated configuration values.

`sudo service postgresql restart`

 

Next, create a new rule on the UFW firewall to allow connection from the 172.16.0.0/12 private subnet to connect to the port 5432 :

`sudo ufw allow from 172.16.0.0/12 to any port 5432`

 

Now we want to get the IP address of the host, which is what the docker containers use to connect to the host. (127.0.0.1 doesn't work here, as using this means refering to the docker container itself, not the actual host, unless you change some network mode which I find it kinda complex).



On the host machine, run `ifconfig` to get the network information. You should see there's a docker0 adaptor :

```bash
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
ether 02:42:bd:6f:e6:46  txqueuelen 0  (Ethernet)
```

 

The **172.17.0.1** is the IP address of the host from the eyes of docker containers.



To connect to the host's database, inside docker container, use this postgreSQL URL : 

```
postgres://db_user:db_password@172.17.0.1:5432/db_name
```



That's it!

Disclaimer: Not claiming this is best practice or anything, I have just learned about Docker and Docker Compose for few days before writing this post out of frustration.

<script async data-uid="d862c2871b" src="https://rubyyagi.ck.page/d862c2871b/index.js"></script>