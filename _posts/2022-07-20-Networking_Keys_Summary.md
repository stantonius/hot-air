---
title: "Networking Notes"
description: "I knew nothing about networking despite using them every day. Now I know slightly more than nothing"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [networking, computer_science]
---



# 2022-07-20-Networking_Keys_Summary

I just spent a bunch of time reading up and watching Youtube videos on networking (because when wokring with Proxmox, there is an assumption that you understand all of these concepts, which I do not). So below is a summary of what I have picked up.

### Notes
This is an evolving document as I inevitably pick up more setting up this home server

#### IP Address
Unique identifier of your device or network (analogous to a street address of your home) within a network, such as 
* Always 4 bytes long (where each byte has 8 bits, called an *octet*)
* Humans were short-sighted in creating IPv4 address range (far too small) and overzealous in distributing blocks of IP addresses to anyone who wanted some (eg. Fortune 500 companies in the 80s), therefore we have a shortage of Public IP addresses that require patch solutions (Private IPs, DHCP, NAT)

Take the following addresses as examples:
* *Public IP*: `25.87.99.204`
* *Private IP*: `192.168.0.133`
	* The `192.168.0` is the **network address** and `.133` is the **host address**
	* How do we know which is which? For most people dealing with their home networks, this pattern can just be accepted. However the actual way we can tell which part is which is by looking at the subnet mask (see below)

#### Subnet
A network within the larger network - a network within a network. It is usually a small subset of IP addresses, where hosts within the subnet can communicate with eachother without traversing "the internet" using the *Private IP address ranges defined by the subnet mask*. A subnet has one or many **gateways** which are used to call other Public IP addresses.
* Can you create multiple subnets on your home network? Yes. Can hosts on Subnet A communicate to hosts on Subnet B without a router? No. You would need a second router to add a new subnet, or a single industrial router that allows for subnetting configuration. But the simple answer is: hosts can only communicate within their subnet without involvment from the gateway.

#### Subnet Mask
Defines the IP address ranges within a subnet. The standard subnet mask `255.255.255.0` for a private home network - a cheap way to may sense of this is any octet with a value of `255` is unavailable to use in our subnet. Because of how base-2 numbers work, if we only have 1 Byte to work with, that means that for a typical subnet we can have a maximum of 253 unique IP addresses (cannot use `.0` or `.255` and `.1` is traditionally the gateway address)
* Note: the last number in the subnet (`255`) is the **Broadcast ID** - anything sent to this ID is *broadcast* to *all devices on the network*
* Below are other less common subnet masks along with their **CIDR** notation (which is a faster way of writing the subnet mask)
	* **Mask length** denotes *how many bits out of 32 are occupied/unavailable*. This is why the final octet in the subnet mask jumps from a  `0` to `128` when increasing the mask length from 24 to 25: 32 - 25 = 7. 2^7 = 128

| Subnet mask | Mask length (aka CIDR notation) | Available addresses |
|:--------------:|:---------------:|:---------------------:|
| 255.255.240.0 | 20 | 4096 |
| 255.255.254.0 | 23 | 512 |
| **255.255.255.0** | **24** | **256** |
| 255.255.255.128 | 25 | 128|
| 255.255.255.192 | 26 | 64|
|255.255.255.254| 31 | 2| 


> Remember how to count in binary - line up the sequence 2^0 to 2^7 and count each product in this sequence if it has a 1 in that place

#### LAN
Your Local Area Network aka *your home subnet*; for most people represented with the standard subnet mask of `255.255.255.0`

#### WAN
The Wide Area Network (basically, the internet). Although the WAN is managed by the ISP - if you log into your router, the WAN IP is your Public IP (the same IP address returned when querying [whatismyipaddress.com](https://whatismyipaddress.com/)). The WAN Default Gateway is your ISPs gateway to the broader internet (netwokrs within networks)

#### DHCP
The (auto) process for **assigning** *hosts* (any device connected to the network) an IP address within the network. Usually good to use as default since it avoids assigning duplicate IP addresses

#### Host
Any device connected to a network

#### Gateway
a.k.a. **a router**; the entry and exit point(s) within a network. By convention, it is often the first node in the subnet (the `.1` in the final octet following the *network address*)
* It is the routers job to map packets to/from the public internet to the local, private IP addresses of the subnet, because Private IPs are not part of the public-facing packet data. This is done via a *routing table* stored within the router.
* I have seen routers with addresses of `192.168.0.1` and `10.0.0.1`
	* Note you access the router admin portal using the same IP address as a packet does

#### NAT
Stands for Network Address Translation; converts one IP address to another; is considered a "band-aid" solution for the IP address availability panic. Performed in a gateway (eg. your router), this is how your Private IP address in your local network can use a single Public IP address and still receive the packets back to that specific device. If this  didn't exist, either a) every device would have to have a Public IP (which is what this "band aid" solution tried to fix in the first place) or b) every device on your local network would receive all of the requested packets once they were returned via the Public IP address
* When digging deeper, ISPs can also perform NAT (thus your request goes through *two* translations before reaching "the internet"). This is how large ISPs manage a customer base that outgrows its alloted IP range given by IANA

#### DNS
Stands for Domain Name System; the internet's phonebook. A public record of IP address to text (called a **hostname**) mappings (so that you can type `google.com` instead of memorizing `172.217.13.196`)


### Resources
* This amazing series by [NetworkChuck](https://youtube.com/playlist?list=PLIhvC56v63IKrRHh3gvZZBAGvsvOhwrRF)