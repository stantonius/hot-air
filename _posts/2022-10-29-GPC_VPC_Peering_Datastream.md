---
title: "GCP Datastream via VPC"
description: "Using Datastream to transfer our data from CloudSQL to BigQuery using private IP addresses"
layout: post
toc: true
comments: true
image: images/some_folder/your_image.png
hide: false
search_exclude: false
categories: [fastpages, jupyter]
metadata_key1: metadata_value1
metadata_key2: metadata_value2
---


## Objective

1. Send data from CloudSQL to BigQuery via a private IP address using Datastream. 

## Background

We use our GCP-managed CloudSQL instance to quickly ingest event data from our app - this is what traditional databases are designed for. However given the volume of data that we store, running *queries* on this data while keeping it online and available to handle new additons and updates means we are running into performance issues. We could try and scale our CloudSQL VM, but as [this good post](https://medium.com/google-cloud/replicate-data-from-bigquery-to-cloud-sql-2b23a08c52b1) argues, this is neither cost effective nor high performing.

As an alternative, GCP has a purpose-built solution for handling large queries - it is appropriately named BigQuery. The downside of this solution is that it is relatively slow compared to our CloudSQL instance - it takes significantly longer to respond to queries or even small updates.

Clearly there is a system design compromise to be made in terms of cost and performance. After evaluating all possibilities, and given the nature of our app (where immediate analytics is not a priority), we have descided to add in BigQuery for our analytics output. As a result we can safely and cost-effectively manage data uploads while deferring analytics responsibility to the slower BigQuery.

But how do we "mirror" the data from CloudSQL to BigQuery? The current best solution I have seen is to use GCP's Datastream option.

### Why Datastream?

The answer on how to do properly link CloudSQL to BigQuery wasn't immediately clear following a Google search. Fortunately I came across [a talk from last year](https://gdg.community.dev/events/details/google-gdg-san-diego-presents-practical-datastream-cloudsql-to-bigquery-change-data-capture-on-google-cloud/) that discussed DataStream as being the right solution for this task. The definition of DataSteam was presented as follows:

> Datastream from Google is a serverless change data capture and replication service. This allows organizations to replicate data across multiple databases, storage systems and is **especially useful for replicating OLTP (Online Transaction Processing) data in MySQL into an OLAP (Online Analytical Processing) database such as BigQuery**. This talk walks through setting up connection profiles, streams and touch on some useful debugging if things don't go as planned

## Steps

Unfortunately, the setup of Datastream is rather complicated if we want to use a VPS/private IP addresses to transfer the data. Below are the steps that ultimately worked for me - hopefully they save someone else the pain I went through.

Note I will include some 

### 1. Configure the CloudSQL instance for use in Datastream

This step is straightforward. As mentioned [here](https://cloud.google.com/datastream/docs/configure-your-source-mysql-database#cloudsqlformysql), we need to do two things: enable binary logging and create a datastream user using the commands they outline.

The important point to realize here is that we are creating a **new user** called `datastream`. In this step you provide a new password. The username and password from this step are **entered in the Datastream private connection setup**.

### 2. Create a Cloud SQL Proxy VM

This is one of the confusing steps for me. [The docs](https://cloud.google.com/datastream/docs/private-connectivity) describe the overall reasoning for needing a Cloud SQL Proxy to connect to our CloudSQL instance. However there are a few takeaway points that I picked up here.

1. The CloudSQL instance doesn't *really* sit within our VPC. Despite us assigning the Private IP connection when creating/editing our CloudSQL instance, it technically sits on another VPC that Google manages behind the scenes. 
2. The Datastream setup *also uses its own VPC*. As such, we are ultimately creating a **VPC peering** setup between our VPC and the Datastream VPC.
3. I got confused by the diagram in the documentation that suggests there is a proxy server that "sits in front" of the CloudSQL instance. There may be, but this is not something we need to worry about

![sqlauthproxy_confusion.png](https://storage.googleapis.com/craigstanton-public-assets/images/vpc/sqlauthproxy_confusion.png)


To simplify - the Datastream connection to CloudSQL must pass through 3 VPCs. It is for this reason that we need a CloudSQL proxy.

As a result, we need to create a VM that sits in our **main VPC**. This can be the smallest VM that GCP allows. 

#### A note on Service Accounts

Every VM that is created also has a default Service Account. This service account needs to have the  `Cloud SQL Client` role in order for this VM to function as a Cloud SQL Proxy agent. 

What I ended up doing was creating a bespoke service account that explicilty has the `Cloud SQL Client` role *before* I created the VM. Then during the VM setup, I selected this SA to be the default service account of the VM. This allowed me to down load the keys for this SA and upload them to the Cloud SQL proxy instance (which is easily done if you SSH into the VM from the GCP console - there is a button at the top of the SSH window that says `Upload Files`). 

Note that there are others way to authenticate such that the VM is enabled as a Cloud SQL proxy - te method above was just the one I chose.

#### Configuring the Cloud SQL Proxy

Once a tiny VM is spun up, we can SSH into the machine and run the commands outlined [here](https://cloud.google.com/sql/docs/mysql/connect-admin-proxy#install) in order to install the proxy binary (a runable program). Then to run the proxy, we enter:

```bash
  ./cloud_sql_proxy -instances=INSTANCE_CONNECTION_NAME=tcp:0.0.0.0:3306 -credential_file=PATH_TO_JSON_KEY \
	  -ip_address_types=PRIVATE
```

Note:
* `-ip_address_types=PRIVATE` is required if there is a Public IP on your CloudSQL instance
* `-credential_file=PATH_TO_JSON_KEY`: is the service account key key we uploaded via the console SSH window described bove
* `INSTANCE_CONNECTION_NAME`: this is the **instance name** of your database, not its IP
* `=tcp:0.0.0.0:3306`: really important to write exactly like this (no spaces), the exception being changing the port for a different database type

Once you start this command, it will run in the background indefninitely

### 3. Set up the Datastream Connection

The documentation on how to do this is [here](https://cloud.google.com/datastream/docs/create-a-private-connectivity-configuration). The tricky points that I want to highlight are:

#### Create Private Connectivity

In this step you are asked to specify an IP range; **this can be any IP range you want** that a) is within your main VPC and b) is not being used by anything else (including subnets).

This took me a while to understand - by selecting an available IP range within your VPC here, GCP *automatically* creates a **peering connection** between Datastream and your VPC *using this IP range*.

This IP range is important to note also when troubleshooting firewall access - you may need to configure your firewall to allow traffic from this new range of IPs

#### Create Connection Profile

Next, when creating the Connection Profile, you are asked for the connection details. Make note of the comments below:
* The `hostname` is the **internal IP address** of the **Cloud SQL Proxy VM** we set up in Step 2
* The `port` is `3306` - which seems bizarre because that is the port on the CloudSQL instance and not on the VM. But remember this Cloud SQL Proxy is effectively a *router* that forwrds requests to their destination and port.
* The `username` and `password` is the **datastream user** we created in Step 1

Finally, the last step in setting up a Connection Profile asks if you want to test the connection. While you can do this, make sure that a) the proxy is up and running and b) the firewall is configured. Having this test connection step here was misleading for me as I had fixes to make in the firewall setup before a connection was possible

### 4. Firewall Setup

As a reminder, we now have a new set of IPs created for network peering from the Datastream, through the Cloud SQL Proxy VM, and onto the Cloud SQL instance. As such, we need to ensure that the firewall for our main VPC:

* Allows ingress traffic **from the datastream peering IP range** we specified above **to the Cloud SQL proxy private IP**
	* Make sure that if you specify a port, that ingress has access to port `3306` (or whatever the database port is)
* Also ensure that the private IP address of the Cloud SQL Proxy can reach the CloudSQL instance

## Troubleshooting

If you get an error like the following `(2003 "Can't connect to MySQL server on '192.168.x.x' ([Errno 111] Connection refused)")`, it can be for any of the following reasons:

* The username and password in the Datastream connection is wrong
* The CloudSQL proxy isn;t running with the right parameters
	* Be careful that you have the `=tcp:0.0.0.0:PORT` written exactly like that
	* You have the private IP flag set if there is a public IP for your CloudSQL instance
* The firewall rules are misconfigured
	* Changes to firewall rules seem to take a while to propogate through the communication change. Therefore after changing and still encountering errors, **reset the Cloud SQL proxy VM and restart the proxy**




