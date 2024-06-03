<h1 align="center">
  Hello! 👋
</h1>

This repository is the home for my infrastructure documentation. Here I will attempt to detail my homelab infrastructure, particularly the media automation workflow and corresponding scripts.

## Table of Contents

1. [About the Project](#about-the-project)
2. [Secondary Goals](#secondary-goals)
3. [Notes/To Do](#notesto-do)
4. [Network Architecture](#network-architecture)

## About the Project

As my home network and homelab have grown over the past year, so too have their maintenance requirements and technical complexity. In the event of a catastrophe (say, me breaking my router config 🙃), it becomes troublesome to rebuild exactly without some blueprint or notes. Similarly, as I learn more IaC concepts and transition towards maintaining my infrastructure though IaC practices and tooling, it will be helpful to have some notes detailing the individual components as they relate to the whole. The goal of this project is to document my infrastructure holistically, both for posterity and as a backbone to improve upon it further as I continue to learn.

## Notes/To Do

I've just created this project and as such there's a long list of things I would like to accomplish, but to start:

- ~~create diagrams for network architecture and media workflow~~
- Create write-up or README for network architecture (in this repo)
- Create write-up or README for media workflow (maybe also in this repo?)
- Tie in other repos/projects as they relate to the broader picture here:
  - individual scripts (LFTP, MQTT, Webhook, etc.)
  - Terraform, Ansible, Docker Compose, etc.
  - Caddy? EdgeRouter? If it makes sense to document while preserving secrets

## Network Architecture

![Diagram of network architecture](/assets/ArchV1.png)
Further documentation pending...

## Media Workflow

![Diagram of media workflow](/assets/MediaWorkflowV1.png)
Further documentation pending...
