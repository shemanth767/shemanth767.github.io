---
title: Replacing Google Drive
date: 2022-07-31 19:00:00 +0530
categories: [Programming]
tags: []     # TAG names should always be lowercase
math: true
mermaid: true
toc: true
published: false
---

# Overview

This article summarizes my descent into insanity and the following events that transpired on the fine day of 30th
July 2022. More formally, as I had mentioned in my previous blog post, I wanted to replace Google Drive and Photos
with something self hosted and OSS. It was also the right time as I was bumping against the 15GB free limit on my
Google Drive. I've had my eye on [Nextcloud](https://nextcloud.com/) for a while now, as I've heard it to be
reliable and easy to setup.

My goals were,

1. Encrypted and automated backups
2. Security
3. Responsiveness
4. Cost Effectiveness

# Preparing my existing server

I've been using a Digital Ocean droplet for the past 2 years to host a Bitwarden password manager and some ad-hoc
testing for ongoing projects.

![](/assets/img/nextcloud-post/previous-do-server.png)
_Digital Ocean server configuration_

Couple of things to note,
* **TOR1** is a datacenter in Toronto, Canada. It's pretty far network-wise from Hyderabad. However, this wasn't a
    concern for Bitwarden as the mobile app and extension caches all passwords frequently.
* **Memory of 2GB** - The absolute minimum required for installing Bitwarden.
* **50GB Disk** - This was originally only 25GB, because why would a password manager need more?

Okay, first problem to tackle.

**Problem #0: Setting up SSH login to the server**
I've recently moved to a new PC, so I had to enable SSH login from this one. It was a very straight-forward process.

1. Copy over public key from local to remote server's /root/.ssh/authorized_keys
2. Restart ssh daemon, `service sshd restart`

**Problem #1: Change the domain name of the server**

I've setup an A record on Bluehost, where I bought my domain, to point "verylongname.saihemanth.com" to the static IP
of my server. I wanted to change it to "short.saihemanth.com" instead. Updating it on Bluehost took only 2 mins, and
the DNS record was published in less than 5 mins.

![](/assets/img/nextcloud-post/short-domain-dns-record.png)
_DNS Record for new short domain name_

Bitwarden was being a pain in the ass for switching domain names, more specifically the in-built certbot. It was supposed
to be a single config change in `/opt/bitwarden/bwdata/config.yml` and `./bitwarden.sh restart`. This worked, but the
generated SSL certificates were still linked to the old domain.

I thought, "Okay, no problem. Let me just trigger the renew certificate command. It should pick up the new domain. Right?"
Wrong!

![](/assets/img/nextcloud-post/certbot-renew.png)
_Certbot proceeded to use the old URL!_
