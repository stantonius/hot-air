---
title: "Proxmox GUI access with Tailscale VPN"
description: "Let the professionals handle the access to my home lab"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [tailscale, proxmox, home_lab, smart_home]
metadata_key1: metadata_value1
metadata_key2: metadata_value2
---

## Objective
I wanted to be able to access my Proxmox Home Lab from outside my home. However I am super paranoid about exposing anything to the open internet, especially when I have only a bit of firewall setup experience. This meant I would not set up port-forwarding on my router and SSH into my machine via WAN. From research, the only real option left was a VPN.

I chose to use a managed service from Tailscale.

## Choosing a VPN solution
There are lots of options out there. From memory, some of the solutions I heard about in forums were pfSense, Wireguard, Tailscale, Zerotier. A lot of info came from the [/homelab](https://www.reddit.com/r/homelab/) subreddit and interestingly, one of the main criticisms of a solution like Tailscale is that it goes against the ethos of a home lab, where everything is configured by the tinkerer. While this makes sense in theory, I imagine a lot of these comments came from those far more comfortable with their cybersecurity knowledge than me. Therefore because of the importance of having good security, I decided to stick with Tailscale VPN to have them manage the E2E networking without having to touch the router or the firewall.

## Tailscale
Tailscale is extremely user friendly and is based on the new industry standard (well this is my interpretation anyway) of VPN encryption Wireguard. Why not just go with Wireguard? Well there is some manual setup for this as well, and again given the security nature of this setup criteria, going with Tailscale means I can set-and-forget.

Additionally Tailscale also automatically restarts after the server reboots (I found several forum posts confirming this and also tested it a few times to ensure it is working)

## Adding Tailscale to Proxmox
Once I signed up for Tailscale and downloaded the App to my Mac (which becomes the first *node* in your Tailscale network), it was time to add my Proxmox Home Lab server as my second Tailscale node.

### Secure Access to Proxmox GUI
The first use of Tailscale in my home lab way to provide access to the Proxmox GUI, which exists on my main Proxmox node. This main node is of course Debian Bullseye based OS. The Tailscale Debian installation instructions are [here](https://tailscale.com/kb/1038/install-debian-bullseye/). As soon as you authenticate with Tailscale, your server and its VPN IP address shows up on your Tailscale portal. You can simply copy this IP address and access your Proxmox GUI without having to touch any router or firewall. Amazing.

Given this system is now a server, I also followed their documentation to [turn off key expiration](https://tailscale.com/kb/1028/key-expiry/).

#### Gotchas
1. I will take to my grave how much time it took me to realize this, however you *should* install Tailscale **on your main Proxmox node**, not in a host VM or LXC. This seems extremely obvious now, since the purpose of this is to see the GUI *which only exists on this main node*. However I was trying to keep this main node super clean without addtional installs, and I had misread that I could access this via an LXC (again, way too complicated if it is even possible at all).
2. Make sure to prefix your Tailscale IP address with **`https://`** and add the port **`8006`** to the end - something like: `https://100.100.100.96:8006`. Without these two elements, I could not connect via the browser.

## Summary
So far I have been really impressed with Tailscale. The speed of the connections, cleanliness of the app, the documentation - all is top quality.

Next is to create a Tailscale Proxmox router to handle the various services I want to set up on Proxmox.