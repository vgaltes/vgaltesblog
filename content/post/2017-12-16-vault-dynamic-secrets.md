---
title: 'Dynamic secrets with Vault'
date: '2017-12-17'
categories:
- devops
tags:
- vault
- security
- mysql
lcb: "{{"
rcb: "}}"
---

In the [previous](http://vgaltes.com/devops/vault-basics) article we saw how we can configure [Vault](https://vaultproject.io) and write and read static secrets. Today, we're going to see how we can use Vault to generate temporary users on a MySQL server so we can control access in a more secure way.

First of all we'll need a MySQL server connected to the same network than the Vault server. Let's change the *docker-compose.yml* file to accomplish this.

```
version: '3'
 
services:
  mysql:
    image: mysql
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: "mypassword"
      MYSQL_DATABASE: "test"
    ports:
      - 6603:3306
    volumes:
      - ./etc/mysql/mysql_data:/var/lib/mysql
    restart: always
    networks:
      - vault_mysql_net
 
  vault:
    depends_on:
      - mysql
    image: vault
    container_name: vault.server
    ports:
      - 8200:8200
    volumes:
      - ./etc/vault.server/config:/mnt/vault/config
      - ./etc/vault.server/data:/mnt/vault/data
      - ./etc/vault.server/logs:/mnt/vault/logs
    restart: always
    networks:
      - vault_mysql_net
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_ADDR: http://127.0.0.1:8200
    entrypoint: vault server -config="/mnt/vault/config/config.hcl"
 
networks:
  vault_mysql_net:
    driver: bridge
```

What are we doing here is add another container based on the mysql image and put them on the same network. To know the IPs of the two containers run *docker inspect mysql* and *docker inspect vault.server*. In my case, the two IPs are *172.18.0.2* and *172.18.0.3* respectively.

## Setting up MySQL
Remember that what are we doing is to setup Vault so that we can ask it to create a temporary user and password to acces a MySQL instance. We're not setting up MySQL as a storage backend to Vault.

We'll need a user in MySQL able to create new users. Vault, will use this user to create (and drop, after the TTL) the temporary user it will bring us to connect to MySQL. To increase the security of our system, we want that only Vault could connect using this user. Let's configure MySQL first.

Connect to MySQL using the root account. As we're exporting the MySQL port, we can do this from our laptop:

```
MacBook-Pro:TestVault vga$ mysql -uroot -pmypassword -h 127.0.0.1 -P 6603
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.20 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

Let's create a test user that can only connect from Vault server:

```
mysql> CREATE USER 'test'@'172.18.0.3';
Query OK, 0 rows affected (0.02 sec)
```

Set a password for that user:

```
mysql> set password for 'test'@'172.18.0.3' = 'test';
Query OK, 0 rows affected (0.00 sec)
```

And now, grant privileges to the user. As we want Vault to just create readonly users (in this example), just give it the create user and select privileges:

```
mysql> grant create user, select on *.* to 'test'@'172.18.0.3' with grant option;
Query OK, 0 rows affected (0.00 sec)
```

We're done with MySQL. Let's move to Vault again.

## Setting up the database backend

To be able to create dynamic secrets, we need to setup the database backend. So, let's login to Vault as an admin:

```
MacBook-Pro:TestVault vga$ vault auth -method=userpass username=admin password=abcd
Error making API request.

URL: PUT http://127.0.0.1:8200/v1/auth/userpass/login/admin
Code: 503. Errors:

* Vault is sealed
```

Ooops! As we discussed in the previous article, every time we restart Vault it becomes sealed. Unseal it and login as admin again. Now it's time to mount the database backend. To do that, run the following command:

```
MacBook-Pro:TestVault vga$ vault mount -path=mysql1 database
Mount error: Error making API request.

URL: POST http://127.0.0.1:8200/v1/sys/mounts/mysql1
Code: 403. Errors:

* permission denied
```
Woooot! We don't have permissions to mount the backend. We need to change the policy and rewrite it. Let's change the adminpolicy.hcl file from the previous article to something like this:

```
#authorization
path "auth/userpass/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
}

#policies
path "sys/policy/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
}

#mounts
path "/sys/mounts/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
}

#mysql1
path "mysql1/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
}
```

And rewrite it:

```
MacBook-Pro:TestVault vga$ vault write sys/policy/admins policy=@"adminpolicy.hcl"
Success! Data written to: sys/policy/admins
```

Cool, let's try to mount the database backend:

```
MacBook-Pro:TestVault vga$ vault mount -path=mysql1 database
Successfully mounted 'database' at 'mysql1'!
```

Awesome, we've mounted the database backend in the path mysql1. Now it's time to configure the role we'll be using in this example. Configuring the role we are defining a couple of things: first, the name of the credentials we're going to read in order to create the user. And second, the query Vault will use to create the user. Let's run the following command:

```
MacBook-Pro:TestVault vga$ vault write mysql1/roles/readonly db_name=mysql creation_statements="CREATE USER '{{page.lcb}}name{{page.rcb}}'@'%' IDENTIFIED BY '{{page.lcb}}password{{page.rcb}}';GRANT SELECT ON *.* TO '{{page.lcb}}name{{page.rcb}}'@'%';" default_ttl="1h" max_ttl="24h"
Success! Data written to: mysql1/roles/readonly
```

And finally, we need to configure the connection to the MySQL where we want to create the users. Let's do it:

```
MacBook-Pro:TestVault vga$ vault write mysql1/config/mysql plugin_name=mysql-database-plugin connection_url="test:test@tcp(172.18.0.2:3306)/" allowed_roles="readonly"


The following warnings were returned from the Vault server:
* Read access to this endpoint should be controlled via ACLs as it will return the connection details as is, including passwords, if any.
```

Cool, we're ready to go. Let's read the credentials for the readonly role in mysql1 and see what happens:

```
MacBook-Pro:TestVault vga$ vault read mysql1/creds/readonly
Key            	Value
---            	-----
lease_id       	mysql1/creds/readonly/977e3682-1f02-7165-43d3-919ba4512223
lease_duration 	1h0m0s
lease_renewable	true
password       	A1a-wq10q1qp3v1055up
username       	v-userpass-a-readonly-v5941vstst
```

Wow! It looks like that Vault has created a user and give us its username and password. Let's try to login into MySQL with those credentials:

```
MacBook-Pro:TestVault vga$ mysql -uv-userpass-a-readonly-v5941vstst -p"A1a-wq10q1qp3v1055up" -h 127.0.0.1 -P 6603
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 5.7.20 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

## Summary

Magic!!!! It worked! We can connect to the MySQL instance using the temporary user Vault has created for us. We no longer have to maintain users for different applications or usages, we just need to give our clients access to this endpoint. And thanks to the power of policies, we can give access to this endpoint only to the users we want. Also, the user used to create users, is only accessible via the Vault IP, so it's quite secure.
