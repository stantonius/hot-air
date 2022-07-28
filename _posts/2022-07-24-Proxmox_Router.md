---
title: "Proxmox network routing with NAT and Tailscale"
description: "One approach to remotely connect to VMs on a private local subnet"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [proxmox, smart_home, home_lab]
metadata_key1: metadata_value1
metadata_key2: metadata_value2
---

## Objective

I have set up a Tailscale VPN on my main Proxmox server. Now I want to route traffic to & from all of the VMs and LXC apps in my server without having to install Tailscale on every machine (I know, this is [counter to what they recommend](https://tailscale.com/kb/1019/subnets/), but I am trying to better understand networking so this is a good project).

## Steps
### 1. Create a second Virtual Network Bridge
We need to create a new virtual network bridge that all the subnet host machines will connect to. [This Youtube video]( https://youtu.be/Q5l7VH6b5r4) was excellent in solidifying my plan and understanding. I wanted to do this to both learn about networking but also to avoid having many new devices connected to my home network - while I'm not at risk of using up my allotted 255 local network IP limit, I wanted an easier way to organize all the VMs and containers that will be using this network.

In Proxmox, select your main server node, then *Networking > Create > Linux Bridge*.

![](https://storage.googleapis.com/craigstanton-public-assets/images/proxmox_server/proxmoxVE_2ndbridge.png)

It was important for me to understand that I could **use any subnet range here** when creating this bridge - therefore we are effectively creating a *virtual* Local Area Network (**vLAN**).

> This might not be technically true that this setup is a vLAN. But as far as I can tell, because this is a custom subnet that I configured, any VM or container that uses this `vmbr1` bridge will join this subnet. As such, this is a unique network that was set up with software - hence vLAN.

### 2. Connect additional bridge to the internet

Now that we have created a second Linux bridge `vmbr1`, we need to configure the main Proxmox server node to route traffic from this second bridge to the main gateway. This is done via a Network Address Translation (**NAT**; as a reminder, NAT is the process of allowing private IPs to use the same single Public IP; more on networking can be found [here](https://stantonius.github.io/home/networking/computer_science/2022/07/20/Networking_Keys_Summary.html)).

Configuring NAT in our case involves updating the iptables to bridge traffic from our new network to the internet via `vmbr0` (which is connected to the physical port `enp6s0`). To set up NAT, edit `/etc/network/interfaces` to look like the following:

```bash
auto lo
iface lo inet loopback

iface enp6s0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.0.207/24
        gateway 192.168.0.1
        bridge-ports enp6s0
        bridge-stp off
        bridge-fd 0

auto vmbr1
iface vmbr1 inet static
        address 192.168.99.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s '192.168.99.0/24' -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '192.168.99.0/24' -o vmbr0 -j MASQUERADE
```

The important section to focus on here is everything under `auto vmbr1`.  Everything up to `bridge-fd 0` was added by Proxmox when I created the new bridge in the GUI (and after a reboot). However the last 3 lines are the NAT config that are added manually - these lines are what allow traffic *from* this subnet to reach the internet.

You can check the config of this new network by running `ip address show dev vmbr1`. The output should show the gateway ip address you configured.
```
5: vmbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet **192.168.99.1/24** scope global vmbr1
       valid_lft forever preferred_lft forever
```


### 3. Create VM and connect to new bridge
The next step is to create a VM that is connected to our new `vmbr1` bridge. I chose the [Ubuntu 20.04 server iso](https://releases.ubuntu.com/22.04/ubuntu-22.04-live-server-amd64.iso) as the image for the VM. After running through the GUI setup and setting the disk size to 50GB with 4096MB of RAM, we end up at the *Networking* tab. Here is where we select our new bridge `vmbr1` we created above.

![](https://storage.googleapis.com/craigstanton-public-assets/images/proxmox_server/proxmoxVE_VMbr1.png)


#### Set static IP address for VM
Next we need to start and configure the VM. During the setup, you are asked to configure the network since it cannot be inferred. However there are additonal parameters needed in order to set up the network correctly that cannot be done during installation. Therefore it was easiest for me to *Continue without network*. Don't forget to enable OpenSSH server if asked. 

Once the installation is complete, log in to the VM and edit the file `00-installer-config.yaml` located in `/etc/netplan`.

> The above instructions apply to **Ubuntu distros v18.04+**. For other Linux distros, the network config is probably found in `/etc/network/interfaces`

Because we skipped the networking config during installation, `00-installer-config.yaml` probably looks something like this:
```yaml
network:
  ethernets:
    ens18:
      dhcp4: true
  version: 2
```

Choose your favourite text editor and update the file to add your VMs static IP address (including the network mask slash notation):
```yaml
network:
  ethernets:
    ens18:
	  addresses: 
	  - 192.168.99.99/24
	  gateway4: 192.168.99.1
	  nameservers:
		addresses:
		- 1.1.1.1
		- 8.8.8.8
		search: []
  version: 2
```

Notice that the gateway address is the same as the address we entered when creating our `vmbr1` bridge. Combined with updating our main server node's ip tables, this configuration allows our machine to connect *to* the internet.

How do we allow for incoming traffic to these machines on a separate vLAN?

### 4. Enable port forwarding
In order to SSH *into* our new VM, we have to configure the Proxmox main node to forward any requests to a particular port over *on the main server node* to the `ip.add.re.ss:port` of the VM. We do this with the following command:
```
iptables -t nat -A PREROUTING -i vmbr0 -p tcp -m tcp --dport 10122 -j DNAT --to-destination 192.168.99.99:22
```

If you visit [this gist](https://gist.github.com/basoro/b522864678a70b723de970c4272547c8), there are awesome suggestions on how to map ports in a systemic way. Here for example, the main server port `10122` will map to machine ID `101` on port `22`.

Now SSH'ing into the VM requires one additional port argument as shown below:
```bash
ssh -p 10122 craig@192.168.0.55
```

Notice something funky here:
1. The user (`craig`) is the user *of the VM*, not the user of the main server node
2. The IP address is that of the *main server node*

We have no achieved bi-directional access to our VMs on a custom local subnet - our VMs can access the internet via our NAT setup, and we can access the VMs via port-forwarding via a router on the main server node.

#### Port forwarding with Tailscale VPN
Without any changes, the setup described above would only allow access to the new VM while on the *local network*. However we want to be able to also connect remotely - and the solution I have chosen is to use Tailscale VPN instead of enabling port forwarding or reverse proxy on our *home* consumer router.

We have two options on how to set up our Tailscale VPN. The first, as alluded to in the intro, is to install Tailscale on all of our VMs on our subnet. For various reasons, both practical and stubborn, I decided against this.

The second option was to add Tailscale to our subnet router that we set up above, which is the option I chose. Lucky for us, Tailscale has good documentation that describes exactly how to create a [subnet router](https://tailscale.com/kb/1019/subnets/?tab=linux). There is no point repeating here what is perfectly laid out in the docs, but there are a couple of setup items to be aware of:

1. If you are running the Tailscale router on the main Proxmox server node as I am, you need **both of the arguments below** when starting Tailscale in that server:
```bash
tailscale up --advertise-routes=192.168.99.0/24 --accept-routes
```
where `--advertise-routes=[your_local_subnet]` and the `--accept-routes` argument is required for any Linux machines to listen for the new subnet range (yes, this is sending the message back to itself as our router is also a Linux box). 

2. This may have been obvious to those more familiar with VPN, but to SSH into a machine over a Tailscale VPN, you actually **call the local IP address of the VM** *as if you were on your home network*. For example, instead of calling `ssh -p 10122 craig@100.80.101.92`, where the IP address is the Tailscale IP address for the main server node, you actually call `ssh craig@192.168.99.99`, where the local subnet IP address of your VM. This took me *way longer than it should have to figure out*, and this fact is not mentioned in the docs (maybe because it should be obvious?).

	Furthermore, what also surprised me was that I **didn't have to specify the port** when SSH'ing into the VM with the IP address `192.168.99.99`. As it turns out, the port forwarding iptables rules we configured on the router in the previous section *already handles the port mapping depending on the incoming traffic type*.

	This Tailscale setup also suggests to me that your local subnet range should be slightly unique (ie. still uses a typical 192/10/172 address). By unique I mean the third octet, for example, is a non-traditional number (ie. 99 instead of 0) - because if you are on another network calling the VM address but another device has that same IP address, there will be address clashing. 

To summarize where we are now: we have a Proxmox VM that is connected to a private local subnet, but can reach the internet and is accessible via SSH anywhere in the world (so long as we are logged in to Tailscale).

### 5. E2E test with Docker image
The final test for a complete system is to spin up a Docker image running a server to test that it is accessible via an external (albeit authenticated) device. To do this I will run a default nginx server and access it via a browser. Therefore the first thing I need to do is to at to our port forwarding iptables on our Proxmox main server node. To do this we run the following in the main server shell:

```bash
iptables -t nat -A PREROUTING -i vmbr0 -p tcp -m tcp --dport 10180 -j DNAT --to-destination 192.168.99.99:80
```

Once we know we can reach our test site via HTTP, we can run the following in our new VM shell: 
```bash
docker run --name test -p 80:80 -d nginx
```

Now, while connected via Tailscale (perhaps the best test is via your mobile with the Wifi turned off, just to be extra thorough), enter in your VMs private IP address in your browser. Magically, the default nginx server page welcomes you.


## Additional Notes
Some may argue that we shouldn't be using our main Proxmox server node as the network traffic router. I'm guessing it is to fully segregate responsibilities - leave the main server fully clean and spin up a container router that is purely for network traffic routing. If I notice any problems with the setup described above, I might try this bespoke router container approach - but for now, I just want a working stable setup, which is what I have managed to build.


### Resources
* https://youtu.be/Q5l7VH6b5r4*
* [Running Proxmox behind a single IP address (github.com)](https://gist.github.com/basoro/b522864678a70b723de970c4272547c8)
* [Create Private Network Bridge on Proxmox VE with NAT | ComputingForGeeks](https://computingforgeeks.com/create-private-network-bridge-proxmox-with-nat/)
* [Subnet routers and traffic relay nodes · Tailscale](https://tailscale.com/kb/1019/subnets/)
* [The Digital Life networking video that mentions calling the local IP address over Tailscale](https://youtu.be/kthHizueMiY)
