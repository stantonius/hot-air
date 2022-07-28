---
title: "A slice of internet security"
description: "Overview of security-related items related to my home lab build"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [ssh, ssl, computer_science]
metadata_key1: metadata_value1
metadata_key2: metadata_value2
---

## SSH

| <!-- -->    | <!-- -->    |
|-------------|-------------|
| **What**         | **S**ecure **SH**ell <br> A communication **protocol**|
| **When** | Pretty much all the time (remote access to a computer, secure file transfer, remote automation)
| **Where** | Port 22 (this is the default, but can be any port) |
| **Why** | Its easy (especially on a Mac or Linux where you simply do `ssh craig@ip.add.ress`; Windows you use PuTTy) along with secure |
| **How** | **Client-server model** |

### Deeper dive on SSH How

**Client-server model**: the client initiates connection with the server, which is always listening to **TCP** connection attempts on the ssh port (22). The following is a very basic overview of the steps once an SSH request is triggered:
	1. Both server and client make sure they *can* communicate with eachother (ie. they both use the same protocols and standards). This is usually never an issue if you are using updated OSes
	2. A combination of asymmetric and symmetric encryption mechanisms ensure all future comms between the client and server are encrypted. The beauty of this approach is that the encryption keys are unique for each new connection, meaning any new session has new keys to encrypt the data
	3. Once the above processes are approved, the **user must authenticate themselves**. This can be done either via password or *another pair* of asymmetric keys that identify you to the server.

> Steps 1 & 2 are not something we typically concern ourselves with; these are steps that are handled behind the scenes between the two computers. We usually only engage in step 3 when providing a password

#### User authentication

A great and simple guide [here](https://www.hostinger.com/tutorials/ssh/how-to-set-up-ssh-keys) shows how to create keys and load them to the server for future use. However below are a couple of summary notes:
* Use `ssh-keygen -t rsa` to create a key pair and store it locally.
	* You can add a passphrase for extra security, but then you need to remember this just like a password
* The public `.pub` file is **the only file that should ever leave your local machine**
	* **NEVER** send the private file **anywhere** (it always stays local)
* Run `ssh-copy-id user@serverip` to securely copy your **public key** to the server
	* You will need to log in using a password to do this (possibly as `root` user)


## SSL

> Even though TLS is the modern implementation of SSL, the term SSL is still used colloquially

| <!-- -->    | <!-- -->    |
|-------------|-------------|
| **What**         | Secure Sockets Layer<br> Actually a **legacy** name for the more secure **TLS** (Transport Layer Security)<br> These are **handshake protocols**|
| **When** | Anytime you visit a modern website - it is the **S** in HTTP**S**
| **Where** | HTTPS: Port 80 via TCP<br> TLS: Port 443 |
| **Why** | Created to do the following:<br> 1. Encrypts the HTTP data so it is unreadable to anyone except the proper recipient <br> 2. Validates the website is who they say they are|
| **How** | Asymmetric encryption |

### Deeper dive on SSL How

At risk of oversimplification, below is a summary of what I have picked up regarding SSL.

* An SSL certificate is formatted in the **x509** standard
* The x509 SSL certificate standard includes several attributes, one of which is a **public key** used for asymmetric encryption
* When a client sends an HTTP request (ie. a GET request) to a server, the server first **sends back its x509 SSL certificate**
* The client then:
	1. Validates the certificate came from a verified authority and the domain name is valid
	2. Initiates the **TLS handshake process**, which includes encrypting data with the public key
* Once the TLS handshake is made, session keys are generated and the SSL certificate is no longer read until the end of the session (this speeds up the data exchange after this initial verification is complete)
	* According to [Cloudflare](https://www.cloudflare.com/en-ca/learning/ssl/transport-layer-security-tls/), this TLS connection protocol *in theory* adds more time when the webpage first loads, but in practice new methods improve the process such that the latency is negligible

### Impact on a home lab

While it is interesting to know how HTTPS works anyway, there are some real applications for us when discussing home labs and servers:

1. Creating [self-signed certificates](https://youtu.be/VH4gXcvkmOY) which gets rid of those annoying browser warnings when we are accessing our own private services (like the Proxmox server GUI)
2. [Enabling Portainer](https://www.the-digital-life.com/portainer-multiple-hosts/) to manage Docker containers remotely
3. [Securely monitoring Proxmox stats](https://www.the-digital-life.com/proxmox-monitoring/)

Being comfortable with the above secure connection concepts is crucial to being a systems and networking engineer. It also makes our jobs a lot easier - no passwords and trusted API access is a nice outcome for a few extra lines of config when first establishing connections.
