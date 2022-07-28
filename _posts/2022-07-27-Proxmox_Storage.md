---
title: "Storage in Proxmox"
description: "A brief overview of storage options in Proxmox"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [proxmox, home_lab, smart_home]
metadata_key1: metadata_value1
metadata_key2: metadata_value2
---

Storage decisions in a Proxmox home lab has been the hardest decision for a newbie like me. It seems like everyone already fully understood the storage types, pros and cons, and had bought the appropriate hardware for their use case. There wasn't much out there that discusses the storage in a way I could fully digest. 

It turns out, the reason I struggled so much to understand storage setup was because you need to search for storage setup recommendations *based on your setup and needs*. For example, most people setting up a home lab/server are doing it **primarily as a NAS**. This means that most are looking for help when they have *multiple drives* and are concerned with data *redundancy* - for them, searching for "Proxmox NAS storage options with multiple SSDs/HDDs" will get answers that aren't particularly useful for me who is doing a home lab focused on IoT/smart home applications with hardware that is not condusive to ZFS-redundancy management (1x 500GB NVME, 1x 1TB SATA SSD, 1x 4TB HDD). 

There are a wide range of use cases for a home server that, as mentioned, will determine your ideal storage configuration. These include (but not limited to):
* A home media server
* A gaming server
* Video and photo storage and editing
* Serving a website or app
* IoT/smart home (this is where mine sits)

Unless you are prepared to spend tonnes of money on drives, memory, and a super-server CPU, your home lab cannot be all of these at the same time. Therefore in order to properly configure your storage, you need to know what the main function of your home lab will be. And this is quite frustrating as a newcomer, because often times we buy the hardware before we truly know what we want this system to be.

## My Setup

For context, I have converted a gaming PC into my home lab. The purpose of the home lab is to run containers and VMs to **manage smart home services** - basically, it is an experimentation environment for coding. Eventually it will also serve as a machine-learning cloud environment, where I can run GPU-enabled deep learning experiments. I don't watch many movies or TV shows, and I don't do any video recording and editing (yet, maybe I am the next big Youtube star? Just need to start Youtubing...)

The specs for my machine are:
* **CPU**: AMD Ryzen 7 5800X 3.8 GHz 8-Core Processor
* **RAM**: Crucial Ballistix RGB 32 GB (4 x 8 GB) DDR4-3600 CL16 Memory
* **Storage**:
	* Samsung 970 Evo 500 GB M.2-2280 NVME Solid State Drive
	* Western Digital Black SN750 500 GB M.2-2280 NVME Solid State Drive
	* Samsung 870 QVO 1 TB 2.5" Solid State Drive
	* Western Digital Red Plus 4 TB 3.5" 5400RPM Internal Hard Drive
* **Mobo**: Asus TUF GAMING X570-PRO (WI-FI) ATX AM4 Motherboard
* **GPU**: Asus GeForce RTX 3060 Ti 8 GB TUF GAMING OC Video Card
* **PSU**: Gigabyte P750GM 750 W 80+ Gold Certified Fully Modular ATX Power Supply


## My storage conclusion

To summarize, my priorities are:
* Snapshot and backup capability (for the container and VM configurations)
* Data backup of IoT sensor and motion-sensing images
* Data processing speed

Noticably missing here is data integrity and redundancy. I am not planning to store precious home movies and photos, nor am I storing edited and processed videos. None of the data is super critical - yes, it would be annoying if the drive with this data disappeared. However, as it stands now, nothing will be considered irreplacable. 

Therefore, here is where I have landed on storage setup:
* The `local` storage will continue to run the Proxmox hypervisor OS on the Samsung 970 Evo NVME SSD in the `ext4` file format
* A new `zfs` single disk storage option for images, snapshots, and backups.
* A second `zfs` single storage drive on the 4TB HDD to store security camera images and additional system backups

### Thoughts on ZFS
Initially I wasn't going to bother with ZFS given my single drives and what I misinterpreted as high RAM usage. However, amazing posts [like this](https://www.reddit.com/r/Proxmox/comments/o66a8n/comment/h2qqn50/?utm_source=share&utm_medium=web2x&context=3) emphasize that ZFS is still useful for single drives. Of course you miss out on data redundancy and correction, but you **still get a lot of benefits**.

I was also cautious to use ZFS on NVME SSDs given its [higher wearout rate](https://www.reddit.com/r/Proxmox/comments/ss07l4/guide_to_minimizing_ssdnvme_wearout_with_proxmox/) so I am trying ZFS on my SATA SSD for VM and container images, snapshots and backups.

### Setting up ZFS

The best instructions can be found [in this video around the 7 minute mark](https://youtu.be/HqOGeqT-SCA). 

As a summary, in the GUI, select your main server node, then Storage > Create. Make sure to select the 'Add Storage' option. In order to store more types of data, we need to create a **Dataset** and then a **Directory**.

In the main node shell, we create the Dataset by entering something like  `zfs create zfs1ssd/zfsdata1 -o mountpoint=/zfsdata`, where:
* `zfs1ssd` is the zfs pool you created in the GUI
* `zfsdata1` is the name of the dataset you want to create (can call it anything you want here)
* `mountpoint=/zfsdata` is the location of the dataset. We still need to create the directory in the next step

In the Datacenter, select Storage > Add > Directory. Here we specify an id (ie. `zfs1ssd-data`) and provide the mountpoint we just created (`/zfsdata`). You can now select all different types of content to b able to save here.

## Future Considerations

Eventually I do plan to do some video editing, and maybe even start storing my photos and videos locally. Therefore, on my radar is unbuffered ECC memory (ideally 3200MHz if possible) to take advantage of error correction. I will also use TrueNAS in this case as a VM on my Proxmox server.
