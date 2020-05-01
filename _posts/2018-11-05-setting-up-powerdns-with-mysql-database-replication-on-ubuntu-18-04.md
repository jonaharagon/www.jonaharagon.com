---
layout: post
title: "Setting Up PowerDNS with MySQL Database Replication on Ubuntu 18.04"
cover: "/assets/img/2018-11-05-setting-up-powerdns-with-mysql-database-replication-on-ubuntu-18-04/cover.jpeg"
---

I recently had to setup DNS servers for a project, so I figured that today's as good a time as any to start documenting what I do.

The goal of this guide is to have PowerDNS configured with a MySQL backend, and then use MySQL replication to sync information between slave servers.

## Prerequisites

 - At least two servers. I've been using [DigitalOcean](https://m.do.co/c/fb6730f5bb99) since 2014 and I always recommend them, their $5/month offering works great for DNS. But of course, any host supporting Ubuntu 18.04 will do just nicely.

## Installing MySQL (All Servers)

First, we need to install MySQL on both the master and slave servers. This is an easy step: Just check for updates and install MySQL:

```
apt update
apt install mysql-server
```

A new feature in MySQL 5.7 and up on Ubuntu servers: MySQL is setup to use the `auth_socket` plugin for the root user by default, rather than password logins. This makes it easier to manage and more secure overall. We also won't have to configure things like the root password.

## Installing PowerDNS (All Servers)

Now we can install PowerDNS, and the MySQL backend plugin.

```
apt install pdns-server pdns-backend-mysql
```

During this installation, you'll be prompted to configure a database for PowerDNS automatically. **Do not do this.** This generally doesn't work in Ubuntu 18.04, as the root MySQL user no longer has a password, and is buggy overall. We'll configure the database manually later.

The installation process will end with some messages from `systemctl` that look like an error, because without the database setup, `pdns-server` failed to start. This is expected, and we'll fix that next.

## Configuring MySQL (All Servers)

First things first, we will need to create a database for PowerDNS. Login to MySQL:

```
mysql
```

Then, create the database, and a user for PowerDNS to use:

```
CREATE DATABASE pdns;
GRANT ALL PRIVILEGES ON pdns.* TO 'pdns'@'localhost' IDENTIFIED BY '<YOUR PASSWORD HERE>';
FLUSH PRIVILEGES;
```

Of course, replacing `<YOUR PASSWORD HERE>` with your own password, preferably randomly generated. If you don't yet use a [password manager](https://1password.com/), now is probably the time to get started.

Now, we'll need to create a bunch of tables. Just paste the following [schema](https://doc.powerdns.com/authoritative/backends/generic-mysql.html#default-schema) into the MySQL prompt.

```
USE pdns;

CREATE TABLE domains (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255) NOT NULL,
  master                VARCHAR(128) DEFAULT NULL,
  last_check            INT DEFAULT NULL,
  type                  VARCHAR(6) NOT NULL,
  notified_serial       INT DEFAULT NULL,
  account               VARCHAR(40) DEFAULT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB;

CREATE UNIQUE INDEX name_index ON domains(name);


CREATE TABLE records (
  id                    BIGINT AUTO_INCREMENT,
  domain_id             INT DEFAULT NULL,
  name                  VARCHAR(255) DEFAULT NULL,
  type                  VARCHAR(10) DEFAULT NULL,
  content               VARCHAR(64000) DEFAULT NULL,
  ttl                   INT DEFAULT NULL,
  prio                  INT DEFAULT NULL,
  change_date           INT DEFAULT NULL,
  disabled              TINYINT(1) DEFAULT 0,
  ordername             VARCHAR(255) BINARY DEFAULT NULL,
  auth                  TINYINT(1) DEFAULT 1,
  PRIMARY KEY (id)
) Engine=InnoDB;

CREATE INDEX nametype_index ON records(name,type);
CREATE INDEX domain_id ON records(domain_id);
CREATE INDEX recordorder ON records (domain_id, ordername);


CREATE TABLE supermasters (
  ip                    VARCHAR(64) NOT NULL,
  nameserver            VARCHAR(255) NOT NULL,
  account               VARCHAR(40) NOT NULL,
  PRIMARY KEY (ip, nameserver)
) Engine=InnoDB;


CREATE TABLE comments (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  name                  VARCHAR(255) NOT NULL,
  type                  VARCHAR(10) NOT NULL,
  modified_at           INT NOT NULL,
  account               VARCHAR(40) NOT NULL,
  comment               VARCHAR(64000) NOT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB;

CREATE INDEX comments_domain_id_idx ON comments (domain_id);
CREATE INDEX comments_name_type_idx ON comments (name, type);
CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);


CREATE TABLE domainmetadata (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  kind                  VARCHAR(32),
  content               TEXT,
  PRIMARY KEY (id)
) Engine=InnoDB;

CREATE INDEX domainmetadata_idx ON domainmetadata (domain_id, kind);


CREATE TABLE cryptokeys (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  flags                 INT NOT NULL,
  active                BOOL,
  content               TEXT,
  PRIMARY KEY(id)
) Engine=InnoDB;

CREATE INDEX domainidindex ON cryptokeys(domain_id);


CREATE TABLE tsigkeys (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255),
  algorithm             VARCHAR(50),
  secret                VARCHAR(255),
  PRIMARY KEY (id)
) Engine=InnoDB;

CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);
```

This creates the default schema for PowerDNS, all the tables and records it needs to operate. Now, exit MySQL:

```
exit;
```

## Configuring PowerDNS (All Servers)

With MySQL all setup, we'll need to change PowerDNS' configuration. Open the MySQL configuration file first:

```
nano /etc/powerdns/pdns.d/pdns.local.gmysql.conf
```

And add the password for the `pdns` MySQL user we created earlier to the `gmysql-password` line. Save and exit.

Now, make a backup of the default configuration:

```
mv /etc/powerdns/pdns.conf /etc/powerdns/pdns.conf-orig
```

And create a new configuration file:

```
nano /etc/powerdns/pdns.conf
```

Pasting in the following configuration:

```
# Replace ns1.example.com with your master nameserver's hostname
default-soa-name=ns1.example.com
include-dir=/etc/powerdns/pdns.d
launch=
security-poll-suffix=
setgid=pdns
setuid=pdns
```

Additionally, on the **master server only**, include the following:

```
api=yes
# Replace <RANDOM STRING> with a randomly generated key for API access
api-key=<RANDOM STRING>
```

We're enabling API access here, but it will only be accessible via localhost. This is for PowerDNS-Admin, the web client we'll be installing later.

## Disable systemd-resolved (All Servers)

Ubuntu 18.04 has some built-in resolver running on port 53 that will interfere with PowerDNS. Just disable it now.

```
systemctl stop systemd-resolved
systemctl disable systemd-resolved
```

Now, we should be able to start PowerDNS:

```
systemctl start pdns
```

## MySQL Replication Setup

These steps are very important and somewhat complicated, but once it's setup you should never have to touch it again. This setup allows for near instant DNS updates across all your nameservers, and has some benefits over PowerDNS' other replication method, the "supermaster" setup, which can be unstable at times and doesn't remove deleted zones across every server automatically.

### Master Server Configuration

On the master server only, edit the MySQL configuration file:

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Changing the following settings:

```
bind-address = 0.0.0.0

server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 10
max_binlog_size = 100M
binlog_do_db = pdns
```

Then, restart MySQL:

```
systemctl restart mysql
```

Now, we need to create a user for the slave servers, so they can access the database to replicate it. Login to MySQL

```
mysql
```

And create a user for your slave server. An important thing I noticed: I had to set the slave's IP address in the MySQL user (as shown below), I couldn't get replication working with a wildcard. If you want to try using a wildcard host for the user, YMMV.

```
GRANT REPLICATION SLAVE ON *.* TO 'pdnsslave'@'<IP OF YOUR SLAVE SERVER>' IDENTIFIED BY '<REPLACE PASSWORD HERE>';
FLUSH PRIVILEGES;
```

If you have *more than one* slave server, the way I tested it, I repeated this step to create a new replication slave user for each server, changing the username each time. You may be able to use the same user for all servers, but I didn't test that configuration.

Now, we need some important information from MySQL before we can configure our slave servers:

```
show master status;
```

```
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      606 | pdns         |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

Note the file and position values here, in this case `mysql-bin.000001` and `606`. We'll need those numbers later.

### Slave Server Configuration

On the slave server(s) only, open the MySQL configuration file:

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

And make the following changes:

```
server-id=2
relay-log=slave-relay-bin
relay-log-index=slave-relay-bin.index
replicate-do-db=pdns
```

The `server-id` variable needs to be unique for each server, so if you have more than one slave server increase it accordingly, i.e. `server-id=2`, `server-id=3`, etc.

Restart MySQL:

```
systemctl restart mysql
```

Now, we will configure the slave server to login to the master MySQL database and replicate it locally. Enter MySQL:

```
mysql
```

Enter the following. Remember the file and position we noted earlier? You'll be entering those here, as well as the password we created for the slave user.

```
change master to
master_host='<MASTER SERVER'S IP>',
master_user='pdnsslave',
master_password='<PASSWORD>', master_log_file='mysql-bin.000001', master_log_pos=606;
start slave;
```

Now, check the slave status to see if it worked:

```
show slave status;
```

## Conclusion

That should be it! If you see any error messages following that last command, some additional troubleshooting may be required. But otherwise, you should now have MySQL replication.

In the next few days, I'll be posting a guide on setting up [PowerDNS-Admin](https://github.com/ngoduykhanh/PowerDNS-Admin) on your master server for web-based zone configuration. It's really a great tool for zone editing. Stay tuned!

I hope you found this guide helpful! If there's something you feel I missed, please feel free to reach out and let me know.
