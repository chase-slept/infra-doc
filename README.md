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
  - [Wireguard](#wireguard)
  - [Caddy Server](#caddy-server)

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
(You may want to click to expand the image for better viewing.)

### Local Network

On the right-hand side of this diagram is my local network, the primary purpose of which is to provide users ad-free web browsing, serve media through a locally-hosted media server, and provide stable and secure Smart Home automation features. Most of the networking is handled by an EdgeRouter X---it handles DHCP reservations and forwards DNS requests to an ad- and malware-blocking Pihole DNS server running on a separate Docker network. There are also a few static routes and other rules configured on the ERX which allow for it to complete routes to a few Docker proxy networks and an external Wireguard VPN Tunnel. There are several IoT hubs and devices that make up a rudimentary Smart Home setup, mostly handled by Apple Home (with some HomeAssistant and Homebridge components); I expect to split this network up as the Smart Home expands later on. Most services on the local network serve clients like AppleTVs, mobile devices, laptops and computers. Hosting these services and microservices are servers like a Raspberry Pi running DietPi, a NAS running TrueNAS Scale, and an older Dell ProBook laptop running Proxmox. Most services are running as Docker containers, hosted on the Raspberry Pi. The local media servers (Jellyfin and Audiobookshelf) run as Kubernetes containers on the TrueNAS host, though they are mostly pre-configured so I won't go into detail about helm charts or other K8s jargon I don't really understand yet anyway!

### External Network

To access some of these services on an external network (i.e. over the internet, pictured on the left-hand side of the diagram), I've configured a Wireguard VPN tunnel on an external server (VPS) somewhere on the internet. With one endpoint on my Pi and the other endpoint on the VPS, this creates a secure, encrypted connection between my local network and an external network. To expand on this a bit, the VPS is configured as a reverse proxy using Caddy. It also doubles as a web server and serves a static website. Using a domain name and pointing its DNS records to the VPS's static IP allows me to route traffic easily and securely over the internet to the reverse proxy, through the secure Wireguard tunnel, and then directly to my internal services and applications.

One major advantage to this is that if your ISP is like mine and uses CG-NAT to distribute IP addresses, you can forward traffic 'around' the CG-NAT, bypassing the port-forwarding limitations that CG-NAT introduces to local networks. With a properly fast internet connection, this would allow you to access your local media content over the internet via Plex/Jellyfin/etc. Unfortunately, my internet connection has terrible upload speed, preventing me from accessing anything remotely but audiobooks and other small files. To work around this, I've complicated the external network even further by adding a high-bandwidth server to the mix! This server specifically runs Jellyfin and has a smaller storage capacity, but it has plenty of bandwidth for streaming media files. To enable me to transfer files to and from this server and the local NAS, I've established an SSH Jump through the VPS (now acting as a Bastion server) and on to the Pi through the Wireguard tunnel; another secure, end-to-end encrypted connection. This connection is necessary to facilitate the media workflow that organizes and transfers files across both networks. Check out the [Media Workflow](https://github.com/chase-slept/media-workflow) project for more information on that.

### Wireguard

To secure the connection between the Bastion server and my local network, I've configured Wireguard as the only allowed connection. Any other requests to the Bastion server are dropped by the firewall by default, and any reverse proxy connections through Caddy use the Wireguard interface to encrypt the connection. One challenge I encountered here was the return traffic getting lost on the local network. To resolve this issue, I've set PreUp and PostDown rules on the Bastion endpoint to define important routing rules. This allows our rules to be applied to the interface only when it is enabled and removed when it is disabled. On the Bastion server, the Wireguard configuration and its routing rules look like this (I've sanitized sensitive IPs and values):

```
[Interface]
PrivateKey = <value>
ListenPort = 51820
Address = 172.0.0.1/32

# packet forwarding
PreUp = sysctl -w net.ipv4.ip_forward=1

# packet masquerading
PreUp = iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o wg0 -j MASQUERADE

[Peer]
PublicKey = <value>
AllowedIPs = 172.0.0.2, 10.0.0.0/24, 10.10.10.0/24
PersistentKeepalive = 25
```

The `Address` is our Wireguard interface address and we'll use this in Caddy to reverse proxy though the WG interface. `AllowedIPs` includes the other WG endpoint on our Raspberry Pi, our local network, and our Docker network, so we can likewise connect and reverse proxy to those. Our first PreUp rule enables the sysctl ip_forward value which allows us to forward packets over the interface. The next PreUp/PostDown rule sets an iptables SNAT (or Masquerade) on the Wireguard interface. This defines the return route for our requests after they're forwarded to the local network. These rules would normally be set on the server directly, but since we only really want these to apply on the WG interface, adding them directly into the configuration as Pre/Post rules is a simple way to allow them to apply only to this interface and persist until the interface is closed.

On the client- or local-side, we have a matching configuration file that looks like this:

```
[Interface]
Address = 172.0.0.2/24
ListenPort = 51820
PrivateKey = <value>

[Peer]
PublicKey = <value>
AllowedIPs = 172.0.0.1
Endpoint = <static IP of Bastion Server>:51820
PersistentKeepalive = 25
```

Here the configuration is a little simpler. The `Endpoint` here needs to point to the static IP of our Bastion Server and the `AllowedIPs` incudes only the other end of the tunnel. That's all the configuration required to create our secure Wireguard tunnel.
### Caddy Server

More documentation on this in-progress ...