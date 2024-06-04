<h1 align="center">
  Hello! 👋
</h1>

This repository is the home for my infrastructure documentation. Here I will attempt to detail my homelab infrastructure, particularly the network architecture and [media workflow](https://github.com/chase-slept/media-workflow).

## Table of Contents

- [Table of Contents](#table-of-contents)
- [About the Project](#about-the-project)
- [Notes/To Do](#notesto-do)
- [Network Architecture](#network-architecture)
  - [Local Network](#local-network)
  - [External Network](#external-network)

## About the Project

As my home network and homelab have grown over the past year, so too have their maintenance requirements and technical complexity. In the event of a catastrophe (say, me breaking my router config ... again 🙃), it becomes troublesome to rebuild exactly without some blueprint or notes. Similarly, as I learn more IaC concepts and transition towards maintaining my infrastructure though IaC practices and tooling, it will be helpful to have some notes detailing the individual components as they relate to the whole. The goal of this project is to document my infrastructure holistically, both for posterity and as a backbone to improve upon it further as I continue to learn.

## Notes/To Do

I've just created this project and as such there's a long list of things I would like to accomplish, but to start:

- ~~create diagrams for network architecture and media workflow~~
- ~~Create write-up or README for network architecture (in this repo)~~ mostly done, need to add Caddy repo and documentation
- ~~Create write-up or README for media workflow (moved to a separate repo~~)
- Create/Tie in other repos/projects as they relate to the broader picture here:
  - ~~individual scripts (LFTP, MQTT, Webhook, sync-script, etc.)  moved to separate repo~~
  - Terraform, Ansible, Docker Compose, etc.
  - Caddy/Bastion server configuration
  - EdgeRouter? If it makes sense to document while preserving secrets

## Network Architecture

![Diagram of network architecture](/assets/ArchV1.png)

### Local Network

On the right-hand side of this diagram is my internal local network. Most of the networking is handled by an EdgeRouter X---it handles DHCP reservations and forwards DNS requests to a Pihole DNS server running on a Docker Network. There are also a few static routes and other rules configured in the ERX which allow for it to complete routes to a few Docker proxy networks and an external Wireguard VPN Tunnel. There are several IoT hubs and devices that make up a rudimentary Smart Home, mostly handled by Apple Home (with some HomeAssistant and Homebridge components); I expect to split this network up as the Smart Home expands later on. Most services on the local network serve clients like AppleTVs, mobile devices, laptops and computers. Hosting these services and microservices are servers like a Raspberry Pi running DietPi, a NAS running TrueNAS Scale, and an older Dell ProBook laptop running Proxmox. Most services are running on Docker hosted on the Raspberry Pi.

### External Network

To access some of these services on an external network (i.e. over the internet, pictured on the left-hand side of the diagram), I've configured a Wireguard VPN tunnel on an external server (VPS) somewhere on the internet. With one endpoint on my Pi and the other endpoint on the VPS, this creates a secure, encrypted connection between my local network and an external network. To expand on this a bit, the VPS is configured as a reverse proxy using Caddy. It also doubles as a web server and serves a static website. Using a domain name and pointing its DNS records to the VPS allow for me to route traffic easily and securely over the internet to the reverse proxy, across the secure Wireguard tunnel, and then directly to my internal services and applications.

One major advantage to this is that if your ISP is like mine and uses CG-NAT to distribute IP addresses, you can forward traffic 'around' the CG-NAT, bypassing the port-forwarding limitations that CG-NAT introduces to local networks. With a properly fast internet connection, this would allow you to access your local media content over the internet via Plex/Jellyfin/etc. Unfortunately, my internet connection has terrible upload speed, preventing me from accessing anything remotely but audiobooks and other small files. To work around this, I've complicated the external network even further by adding a high-bandwidth server to the mix. This server specifically runs Jellyfin and has a smaller storage capacity, but it has plenty of bandwidth for streaming media files. To enable me to transfer files to and from this server and the local NAS, I've established an SSH Jump through the Bastion VPS and on to the Pi. This connection is necessary to facilitate the media workflow that organizes and transfers files across both networks. Check out the [Media Workflow](https://github.com/chase-slept/media-workflow) project for more information.
