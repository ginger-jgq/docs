---
title: Manual Deployment
summary: Learn how to manually deploy a multi-node CockroachDB cluster on multiple machines.
toc: false
toc_not_nested: true
---

This page shows you how to manually deploy a multi-node CockroachDB cluster on multiple machines. 

{{site.data.alerts.callout_info}} For testing and development, you can <a href="start-a-local-cluster.html">Start a Local Cluster</a> or <a href="cloud-deployment.html">Deploy on GCE or AWS using Terraform</a>. You can also <a href="http://uptimedba.github.io/cockroach-vb-single/cockroach-vb-single/home.html">Run CockroachDB inside a VirtualBox VM</a> (community-supported).{{site.data.alerts.end}}

<div id="toc"></div>

## Requirements

This process assumes the following:

- You have the [CockroachDB binary](install-cockroachdb.html).
- You have SSH access to each machine. This is necessary for distributing binaries and, in the case of a secure cluster, certificates. 
- Your network configuration allows the machines to talk to each other and clients to talk to the machines.

## Recommendations

- Put each node on a different machine. Since CockroachDB replicates across nodes, placing more than one node on a single machine increases the risk of data unavailability when a machine fails.  
- Run [NTP](http://www.ntp.org/) or other clock synchronization software on each machine. CockroachDB needs moderately accurate time; if the machines' clocks drift too far apart, transactions will never succeed and the cluster will crash. 

## Deploy an Insecure Cluster

### 1. Set up the first node

Copy the `cockroach` binary to the first machine and then start the node:

~~~ shell
$ cockroach start --insecure --host=<node1-hostname>
~~~

This command sets the node to insecure and identifies the address at which other nodes can reach it. Otherwise, it uses all available defaults. For example, the node stores data in the `cockroach-data` directory, listens for internal and client communication on port 26257, and listens for HTTP requests from the Admin UI on port 8080. To set these options manually, see [Start a Node](start-a-node.html). 

### 2. Set up the second node

Copy the `cockroach` binary to the second machine and then start the node:
    
~~~ shell
$ cockroach start --insecure --join=<node1-hostname>:26257
~~~

The only difference when starting the second node is that you connect it to the cluster with the `--join` flag, which takes the address and port of the first node. Otherwise, it's fine to accept all defaults; since each node is on a unique machine, using identical ports won't cause conflicts.

### 3. Set up additional nodes

Repeat step 2 for each additional node.

### 4. Configure replication

In CockroachDB, you use **replication zones** to control the number and location of replicas for specific sets of data. Initially, there is a single, default replication zone for the entire cluster. You can adjust this default zone as well as add zones for individual databases and tables as needed. 

For more information, see [Configure Replication Zones](configure-replication-zones.html).

### 5. Use the Built-in SQL Client

Start the built-in SQL client from any machine with the `cockroach` binary. This could be one of the node machines or a different machine. 

~~~ shell
$ cockroach sql --insecure --url=postgresql://root@<node-hostname>:26257/?sslmode=disable
# Welcome to the cockroach SQL interface.
# All statements must be terminated by a semicolon.
# To exit: CTRL + D.
~~~

This command uses the `--url` flag to identify the client user and the hostname and port of the node to connect to. You can also specify these details with distinct flags. For more information, see [Use the Built-In SQL Client](use-the-built-in-sql-client.html).

Once you're connected, run some [SQL statements](learn-cockroachdb-sql.html): 

~~~ shell
root@:26257> CREATE DATABASE bank;
CREATE DATABASE

root@:26257> SET DATABASE = bank;
SET DATABASE

root@:26257> CREATE TABLE accounts (id INT PRIMARY KEY, balance DECIMAL);
CREATE TABLE

root@26257> INSERT INTO accounts VALUES (1234, 10000);
INSERT 1

root@26257> SELECT * FROM accounts;
+------+---------+
|  id  | balance |
+------+---------+
| 1234 |   10000 |
+------+---------+
~~~

### 6. Connect an app to the cluster

CockroachDB supports the PostgreSQL wire protocol, so you can use any available PostgreSQL client driver to connect an app to the cluster. 

For a list of recommended drivers that we've tested, see [Install Client Drivers](install-client-drivers.html). For some basic code samples, see [Build a Test App](build-a-test-app.html).  

### 7. Monitor your cluster

The CockroachDB Admin UI lets you monitor cluster-wide, node-level, and database-level metrics and events. To start up the Admin UI, point your browser to the URL in the `admin` field listed in the standard output of any node on startup, for example:

~~~ shell
$ cockroach start --insecure --host=node1.example.com
build:     {{site.data.strings.version}} @ {{site.data.strings.build_time}}
admin:     http://node1.example.com:8080 <-------------------------- USE THIS URL
sql:       postgresql://root@node1.example.com:26257?sslmode=disable
logs:      cockroach-data/logs
store[0]:  path=cockroach-data
~~~

<img src="images/admin_ui.png" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## Deploy a Secure Cluster

### 1. Create security certificates

On a machine that won't be part of the cluster, create the CA, node, and client certificates and keys:

~~~ shell
# Create the CA certificate and key.
$ cockroach cert create-ca --ca-cert=ca.cert --ca-key=ca.key
   
# Create the node certificates and keys.
# For each node, specify any addresses at which the node can be reached.
$ cockroach cert create-node <node1-hostname> <node1-other-hostname> <node1-yet-another-hostname> --ca-cert=ca.cert --ca-key=ca.key --cert=node1.cert --key=node1.key
$ cockroach cert create-node <node2-hostname> <node2-other-hostname> <node2-yet-another-hostname> --ca-cert=ca.cert --ca-key=ca.key --cert=node2.cert --key=node2.key
$ cockroach cert create-node <node3-hostname> <node3-other-hostname> <node3-yet-another-hostname> --ca-cert=ca.cert --ca-key=ca.key --cert=node3.cert --key=node3.key

# Create a certificate and key for each client user, including root. 
$ cockroach cert create-client root --ca-cert=ca.cert --ca-key=ca.key --cert=root.cert --key=root.key
$ cockroach cert create-client <username1> --ca-cert=ca.cert --ca-key=ca.key --cert=username1.cert --key=username1.key
$ cockroach cert create-client <username2> --ca-cert=ca.cert --ca-key=ca.key --cert=username2.cert --key=usename2.key
~~~

Store the CA key somewhere safe and keep a backup; if you lose it, you will not be able to add new nodes or clients to your cluster.

### 2. Set up the first node

Copy the `cockroach` binary, CA certificate, and node 1 certificate and key to the first machine and then start the node:

~~~ shell
$ cockroach start --ca-cert=ca.cert --cert=node1.cert --key=node1.key --host=<node1-hostname>
~~~

This command specifies the location of certificates and the address at which other nodes can reach it. Otherwise, it uses all available defaults. For example, the node stores data in the `cockroach-data` directory, listens for internal and client communication on port 26257, and listens for HTTP requests from the Admin UI on port 8080. To set these options manually, see [Start a Node](start-a-node.html). 

### 3. Set up the second node

Copy the `cockroach` binary, CA certificate, and node 2 certificate and key to the second machine and then start the node:

~~~ shell
$ cockroach start --ca-cert=ca.cert --cert=node2.cert --key=node2.key --host=<node2-hostname> --join=<node1-hostname>:26257
~~~

The only difference when starting the second node is that you connect it to the cluster with the `--join` flag, which takes the address and port of the first node. Otherwise, it's fine to accept all defaults; since each node is on a unique machine, using identical ports won't cause conflicts.

### 4. Set up additional nodes

Repeat step 3 for each additional node. 

### 5. Configure replication

In CockroachDB, you use **replication zones** to control the number and location of replicas for specific sets of data. Initially, there is a single, default replication zone for the entire cluster. You can adjust this default zone as well as add zones for individual databases and tables as needed. 

For more information, see [Configure Replication Zones](configure-replication-zones.html).

### 6. Use the built-in SQL client

You can run the built-in SQL client from any machine with the `cockroach` binary, CA cert, client certificate, and client key. Make sure the machine you want to use has these files and then start the client:  

~~~ shell
$ cockroach sql --url="postgresql://root@<node1-hostname>:26257/?sslcert=root.cert&sslkey=root.key&sslmode=verify-full&sslrootcert=ca.cert"
# Welcome to the cockroach SQL interface.
# All statements must be terminated by a semicolon.
# To exit: CTRL + D.
~~~

This command uses the `--url` flag to identify the client user, the hostname and port of the node to connect to, and the location of the CA certificate and client certificate and key. You can also specify these details with distinct flags. For more information, see [Use the Built-In SQL Client](use-the-built-in-sql-client.html).

Once you're connected, run some [SQL statements](learn-cockroachdb-sql.html): 

~~~ shell
root@:26257> CREATE DATABASE bank;
CREATE DATABASE

root@:26257> SET DATABASE = bank;
SET DATABASE

root@:26257> CREATE TABLE accounts (id INT PRIMARY KEY, balance DECIMAL);
CREATE TABLE

root@26257> INSERT INTO accounts VALUES (1234, 10000);
INSERT 1

root@26257> SELECT * FROM accounts;
+------+---------+
|  id  | balance |
+------+---------+
| 1234 |   10000 |
+------+---------+
~~~

### 7. Connect an app to the cluster

CockroachDB supports the PostgreSQL wire protocol, so you can use any available PostgreSQL client driver to connect an app to the cluster. 

For a list of recommended drivers that we've tested, see [Install Client Drivers](install-client-drivers.html). For some basic code samples, see [Build a Test App](build-a-test-app.html).  

### 8. Monitor your cluster

The CockroachDB Admin UI lets you monitor cluster-wide, node-level, and database-level metrics and events. To start up the Admin UI, point your browser to the URL in the `admin` field listed in the standard output of any node on startup, for example:

~~~ shell
$ cockroach start --insecure --host=node1.example.com
build:     {{site.data.strings.version}} @ {{site.data.strings.build_time}}
admin:     https://node1.example.com:8080 <-------------------------------- USE THIS URL
sql:       postgresql://root@node1.example.com:26257?sslcert=%2FUsers%2F...
logs:      cockroach-data/logs
store[0]:  path=cockroach-data
~~~

<img src="images/admin_ui.png" alt="CockroachDB Admin UI" style="border:1px solid #eee;max-width:100%" />

## See Also

- [Cloud Deployment](cloud-deployment.html)
- [Start a Local Cluster](start-a-local-cluster.html)
- [Run CockroachDB in a VirtualBox VM](http://uptimedba.github.io/cockroach-vb-single/cockroach-vb-single/home.html) (community-supported)
