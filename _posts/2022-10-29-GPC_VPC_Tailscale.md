---
title: "Google Cloud VPC with Tailscale"
description: "Covering the basics of VPC and how to access with a VPN"
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

1. Set up a custom VPC network 
2. Set up a VPN so that we can access our private resources from our local development machines

## Virtual Private Clouds

Out of curiosity, I decided to create a custom VPC for our project. In our case, the specific subnet we are using is in the `northamerica-northeast1` region. We created a subnet range of `10.1.0.0/16`

> Note the range of IP addresses selected for a subnet is completely arbirtrary - in my case there are way too many addresses for this subnet, but I essentially randomly chose the configuration. This is OK because there is currently no charge for static or ephemeral internal IP addresses. However the takeaway point is I could have selected any block of private IP addresses for this project.

<p align="center">
  <img src="https://storage.googleapis.com/craigstanton-public-assets/images/vpc/vpc.png" width="500" height="700"/>
</p>


### Why a Custom VPC network?

Upon reflection, I don't think this was necessary. I think it would have been perfectly acceptable to use the `default` VPC network that GCP creates for every project. The reason I say this is we are not using the VPC to segment access to resources (ie. using vLANs) for security purposes. We are simply using a VPC to prevent public access to private resources. Therefore the `default` VPC would have worked fine for this.

### VPCs are Global

A single VPC is a global resource - it is not tied to any one region. However a VPC itself does not have any IP addresses that are assignable to cloud resources - the only way to add a resource to a VPC is via subnets. 

### VPC Subnets

Subnets are just a segment of IP addresses. You use subnets to create blocks of IP addresses that are assignable to cloud resources.

#### VPC Subnets & Regions

Cloud resources are assigned a private IP address *from a specific subnet*. As a reminder, each subnet must be assigned to a specific GCP region. 

> Can you assign a GCP resource hosted in one region to a subnet in another region? No - the subnet that a resource is assigned *must* be a part of the same region that the resource is hosted in.

The obvious question that arises is: can resources on 1 subnet communicate with a resource on another subnet on the same VPC? The answer all depends on the **firewall rules**. If using the `default` VPC, then the firewall is preconfigured to allow internal traffic between all subnets of a particular VPC. However if we have set up a custom VPC, we need to manually configure the firewall to allow this traffic - this is explained in the [[#Firewall]] section below.

### VPC Access

Using a VPC effectively creates a virtual "wall" around our resources. This means we have full control over access to these resources, and by default access is pretty restricted. Clearly there are many ways to access these resources depending on what we are trying to do, among them are SSH and reverse-proxy setups. However we are not going to discuss those here. 

What I will touch on is how do we access these resources from our local development machines using a VPN. For example, say I want to query a SQL database that only has a private IP address - I need some way to allow inbound traffic to the SQL instance. Check out the [[#Tailscale VPN]] section below for more information.

## Tailscale VPN

So if all of our machines are only accessible within the VPC, how do we access the resources on our local machines? The answer is of course via a VPN.

Tailscale has good resources on how to [connect to GCP VPC](https://tailscale.com/kb/1147/cloud-gce/) and importantly how to enable the VM as a [subnet router](https://tailscale.com/kb/1019/subnets/), so there is no point repeating them here. 

However there are a few concepts that I picked up that are worth outlining:

* The tailscale "tunnel" is a VM that listens for UDP on port 41641 (hence the reason for modifying the firewall to enable traffic on that port)
	* While this seems obvious now, its important for me to stress that VPNs are not magic - they still require *some way* through the firewall. It is just that these enabled ports on a firewall for a VPN are a) not traditional like 22 and b) do not require and manual credential exchange authentication.
	*  Therefore if you only ever wanted to access resources on a VPC via a VPN, you would set up the firewall to block all ingress traffic *except for the Tailscale ports*
* The tailscale VM does have a public IP address so that we can connect to it from anywhere.
	* Note that this does not mean that this public IP address is widely used - in fact, we **disable SSH** **and HTTP/S** on this instance so that it cannot be accessed outside of the VPN
	* We only allow UDP traffic on the specific Tailscale port
* Our Tailscale VM is effectively a **router**. As such, the Tailscale instance *must advertise itself* as a router accepting traffic to *certain IP ranges*.
	* This is done via a Tailscale SDK command. For example: `sudo tailscale up --advertise-routes=10.0.0.0/24,10.0.1.0/24`

So what does this mean in practice? Well in order to test our config locally, connect to a VM, etc., I must be logged in and running Tailscale on my Mac.
 
## Firewall

As alluded to above, just because we have created a VPC and assigned cloud resources to it, doesn't mean that all traffic is automatically permitted between these resources\*. We still need to check the firewall rules to ensure that traffic can flow only where we want it.

\* *if you choose to build your subnets manually. I think internal traffic is allowed if you let GCP configure all 36 subnets automatically*

The two rules we need to set up for our use case are a) to allow all traffic within the subnet to connect to all other machines on the subnet and b) allow Tailscale UDP traffic.

### Allowing Internal Traffic within our VPC

<p align="center">
  <img src="https://storage.googleapis.com/craigstanton-public-assets/images/vpc/vpc_firewall.png" width="380" height="400"/>
</p>


As you can see, the setup above allows us to communicate with any other resource on the VPC - because of the `/8` CIDR notation, this rule covers **all subnets** in our VPC. As mentioned, without this rule you cannot even ping another instance on the subnet - the firewall default denies all traffic that we haven't explicitly allowed.

Should I have allowed all traffic to all ports from any instance with a `10.x` IP address? Well because it is an internal LAN/subnet, this is low risk.

The Tailscale firewall setup was equally as straightforward:

<p align="center">
  <img src="https://storage.googleapis.com/craigstanton-public-assets/images/vpc/tailscale_firewall.png" width="350" height="400"/>
</p>


## Assigning Private IPs to GCP Resources

Generally speaking, it is advisable to have the VPC set up *before* you create these resources because it is sometimes hard to add after the fact. Pay particular attention to serverless and managed CloudSQL instances - these need 


## Background Notes

* A Virtual Private Cloud is just a set of private IP addresses. A VPC can have one or many subnets.
	* The VPC itself is a global resource, not bound to any region
	* A VPC doesn't have IP addresses on its own; private IP addresses are allocated via subnets
* A subnet is a *subset* of internal IP addresses
	* As internal/private IP addresses, they must be within one of the IANA IPv4 private network ranges: `10.0.0.0/8`, `172.16.0.0/12`, or `192.168.0.0/16`
* In GCP VPCs, a subnet can only span a **single GCP region**. This is why GCP creates 36 subnets (as of Oct 2022) within the `default` VPC of a project  - one subnet per region
* Cloud resources are assigned a private IP address from one of the VPC subnets.
	* A single resource can only have 1 private IP address
* VPC is advantageous because:
	1. You do not need to expose as many/any resources to the public internet
	2. Lower latency traffic between resources
* There are some caveats and gotchas (there are probably more - these are just the ones I have come across):
	* Special configuration is needed to assign serverless resources to a VPC (this makes sense - they're ephemeral)
	* SSH'ing to a VM without a Public IP but has a Private IP that is part of the VPC needs to be done via Identty Access Proxy
		* This makes **automating config via Ansible more difficult** (an article on how to tell Ansible to use IAP Tunneling is [here](https://binx.io/2021/03/10/how-to-tell-ansible-to-use-gcp-iap-tunneling/))
	* Fully-managed servers (aka CloudSQL) still need a proxy for VPC network peering
		* Discovered this when setting up DataStream, which uses its own VPC and requires peering to access any resources on another VPC