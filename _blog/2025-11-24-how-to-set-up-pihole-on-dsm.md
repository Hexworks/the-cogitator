---
excerpt: "If you ever wanted to have your own DNS Sinkhole and you have a Synology NAS this guide will let you setup one quickly"
title: "Setting up Pihole on your Synology NAS"
tags: [synology, pihole, docker]
author: addamsson
short_title: "Setting up Pihole on your Synology NAS"
comments: true
updated_at: 2025-11-24
---

> If you ever wanted to have your own DNS Sinkhole and you happen to have a Synology NAS then this guide will show you how to set one up from scratch. 📙 This article assumes that you're using a Synology NAS with DSM.

I had my DS923+ for a while but I didn't use its virtualization feature for much (apart from setting up Jellyfin), but lately I've been really annoyed with all the ads that I was seeing. After doing some digging I found out that there is (at least a partial) solution for this problem: A DNS Sinkhole!

What is a DNS Sinkhole you might ask?

> A DNS sinkhole (also called a sinkhole DNS, blackhole DNS, or DNS blackholing) is a defensive cybersecurity technique used to block malicious or unwanted domains. 
> -- Chat GPT

If configured properly it can be used to route outgoing traffic for your selected domains to `/dev/null`. This means that you can disable trackers, ad services, and also anything that you don't want to go out. Let's look at how we can set this up.


## Network Interface

In order to get this working first we need to look at the network interfaces on our NAS. In order to do this you'll need to SSH into your nas, so first allow it (temporarily):

- Navigate to the *Control Panel*
- Select *Terminal & SNMP*
- Check *Enable SSH service

> 📙 Make sure that you **disable** SSH after you're done!

Now you'll be able to SSH into your NAS. You should see something like this:

```bash
❯ ssh addamsson@192.168.144.2
addamsson@192.168.144.2's password:

Using terminal commands to modify system configs, execute external binary
files, add files, or install unauthorized third-party apps may lead to system
damages or unexpected behavior, or cause data loss. Make sure you are aware of
the consequences of each command and proceed at your own risk.

Warning: Data should only be stored in shared folders. Data stored elsewhere
may be deleted when the system is updated/restarted.

addamsson@Obelisk:~$
```

> 📘 The IP of my NAS is `192.168.144.2`, so I'll grep for this, you should use your own IP

Now let's run `ifconfig | grep 192.168.144.2 -B 1`. It will show you something like this:

```bash
eth0      Link encap:Ethernet  HWaddr 90:09:D0:36:CE:2F
          inet addr:192.168.144.2  Bcast:192.168.144.255  Mask:255.255.255.0
```

Write down the name of the interface (`eth0` in my case), we'll use this later.


## Create a macvlan Interface

Now we'll create a macvlan interface that we'll use for our Pihole. While still in your terminal execute the following command:

Variables:

- `interface`	= The interface you wrote down (usually `eth0`)
- `subnet`	= First 3 digits of your router's ip
- `gateway`	= Your router's address

Command pattern:

```bash
sudo docker network create -d macvlan -o parent=<interface> --subnet=<subnet>.0/24 --gateway=<gateway> --ip-range=<subnet>.198/32 ph_network
```

- `<interface>` stands for your network interface, in our case `eth0`
- `<subnet>` is the first 3 digits of your router's ip. In my case it is `192.168.144`.
- `<gateway>` is your router's address

In my case I executed this command:

```bash
sudo docker network create -d macvlan -o parent=eth0 --subnet=192.168.144.0/24 --gateway=192.168.144.1 --ip-range=192.168.144.119/32 ph_network
```

> 📘 Why do we need this macvlan interface? What is a macvlan interface anyway?
> A macvlan interface lets us create a virtual network interface that will have its own ip address. It allows us to masquerade our pihole container as if it was a separate device on the network. This is important because there are possible port collisions between DSM and Pihole (port `80` is a possible one for example).

Now let's verify that the network was created:

```
sudo docker network ls
```

You should see something like this:

```
43e8b9776ac7   ph_network         macvlan   local
```


## Creating the Folder Structure

Pihole needs disk space, and a specific folder structure so let's navigate to *File Station* and under `/docker` create the following folder structure

```bash
pihole/dnsmasq.d
pihole/pihole
```

Now go to *Control Panel*, and under *File Sharing* look at your *Shared Folder* and note what volume your `docker` folder is on.
For me this is `Volume 1`. We'll use this momentarily.


## Creating the Pihole Container

Open *Container Manager*, navigate to the *Project* tab and click *Create*.

Name your project `pihole` and set its path to `/docker/pihole` and select *Create docker-compose.yml*.

Use the following template and paste it into the dialog:

> 📘 Note that we'll need a bridge network because the pihole macvlan network we're using can't communicate with the NAS. If you'll try to use your Pihole as DNS **within your NAS** it won't work otherwise. If you don't want this DNS in your nas you can skip adding the bridge network.

```yml
version: "3"
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp" 
      - "80:80/tcp"
    networks:
      - ph_network
      - ph_bridge   # feel free to skipt his if you don't need DNS in your NAS
    environment:
      TZ: '<your_timezone>' 				            # set your timezone
      FTLCONF_webserver_api_password: '<your_password>'	# pick a password
      DNSMASQ_LISTENING: local
    volumes:
      - '/volume1/docker/pihole/pihole:/etc/pihole'		    # make sure that you're using the appropriate volume (that you wrote down earlier)
      - '/volume1/docker/pihole/dnsmasq.d:/etc/dnsmasq.d' 	# This mounts the folder you created
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
networks:
    ph_bridge:
      driver: bridge
      ipam:
        config:
          # These are fixed (does not depend on your other configuration), and this is only necessary for your NAS to be able
          # to connect to your pihole
          - subnet: 192.168.10.0/24
            gateway: 192.168.10.1
            ip_range: 192.168.10.2/32 # The IP where you want the pihole to be accessible ONLY FROM THE NAS
    ph_network:
      name: ph_network
      external: true                  # We use external: true because we've already configured this network
```

Now start the container. After starting up if you go to *Container Manager* you should see
- The `pihole` container with a 🟢 next to its name
- A new network named `pihole_ph_bridge` that only connects to your container


## Configure pihole as DNS server

Go to *Control Panel / Network* and check "Manually configure DNS server" and set the bridge network IP address (`192.168.10.2`)

> 📘 If you used the macvlan network instead you'd see that the NAS can't communicate with the outside world.

Go to your router, and configure your Pihole as your DNS server using the macvlan address (`192.168.144.119` in my case).

Now if you browse the internet on a connected device you'll be able to see the queries on your Pihole!


