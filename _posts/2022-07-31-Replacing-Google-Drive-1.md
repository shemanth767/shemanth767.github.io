---
title: Replacing Google Drive with NextCloud - Part 1
date: 2022-07-31 19:00:00 +0530
categories: [Server]
tags: []     # TAG names should always be lowercase
math: true
mermaid: true
toc: true
published: true
---

# Overview

This article summarizes my descent into insanity and the following events that transpired on 30th
July 2022. More formally, as I had mentioned in my previous blog post, I was looking to replace Google Drive and Photos
with something self hosted and OSS. It was also the right time as I was bumping against the 15GB free limit on my
Google Drive. I've been following [Nextcloud](https://nextcloud.com/) for a while now, as I've heard it to be
reliable and easy to set up.

My goals are,

1. Reliability
2. Security
3. Responsiveness
4. Cost Effectiveness

![](/assets/img/nextcloud-post/basic-nextcloud-plan.jpg)

# Preparing my existing server

I've been using a Digital Ocean droplet (a Virtual Private Server) for the past 2 years to host Bitwarden
password manager and some ad-hoc testing for ongoing projects.

![](/assets/img/nextcloud-post/previous-do-server.png)
_Digital Ocean server configuration_

Couple of things to note,
* **TOR1** is a datacenter in Toronto, Canada. Toronto is quite far from Hyderabad in terms of network latency,
    which is not a concern for Bitwarden as the mobile app and extension cache the password vault frequently.
* **Memory of 2GB** - The absolute minimum required for installing Bitwarden.
* **50GB Disk** - This was originally only 25GB because why would a password manager need more?

Okay, first problem to tackle.

## Problem #0: Setting up SSH login to the server
I've recently migrated to a new PC, so I had to enable SSH login from this one. It was a very straight-forward process.

1. Copy over public key from local to remote server's /root/.ssh/authorized_keys
2. Restart ssh daemon, `service sshd restart`
3. SSH into the server.

## Problem #1: Change the domain name of the server

I had an A record configured on Bluehost to point "verylongname.saihemanth.com" to the static IP
of my server. Purely for aesthetic reasons, I wanted to change it to "short.saihemanth.com" instead.
Updating the DNS record barely took 10 mins.

![](/assets/img/nextcloud-post/short-domain-dns-record.png)
_DNS Record for new short domain name_

Bitwarden turned out to be a pain in the ass while switching domain names. Specifically, it was the
built-in certbot. It was supposed to be a single config change in `/opt/bitwarden/bwdata/config.yml` and
restart, `./bitwarden.sh restart`. This worked, but the generated SSL certificates were still linked to the old domain.

I thought, "Okay, no problem. Let me just trigger the renew certificate command. It should pick up the new
domain name from config. Right?" Wrong!

![](/assets/img/nextcloud-post/certbot-renew.png)
_Certbot proceeded to use the old URL!_

I deleted all certificate files used specifically by certbot in the hopes that it'll try to pick up the correct
domain name from config next time. That still didn't work. I struggled with fixing this for over an hour!

![](https://c.tenor.com/g_H8p8ecUyEAAAAC/leslie-angry.gif)

With no other resort, I exported all my passwords using the Bitwarden Web UI and uninstalled bitwarden:
`./bitwarden.sh uninstall`. Reinstalling the entire setup and importing my exported passwords took less than
20 mins.

Not elegant, but problem solved!

# NextCloud

## Problem #2: Setting up Nextcloud

I took a backup of the droplet to start with — in case I brick the server. DigitalOcean bills me for keeping backups too.
Thanks, DO 👍

1. Installing NextCloud itself was very simple. I just had to follow nextcloud all-in-one's [README](https://github.com/nextcloud/all-in-one).
    What does *all-in-one* mean? I didn't know, I only needed the cloud storage functionality, and AIO was easy to set up.
```sh
    sudo docker run -it \
    --name nextcloud-aio-mastercontainer \
    --restart always \
    -p 80:80 \
    -p 8080:8080 \
    -p 8443:8443 \
    --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
    --volume /var/run/docker.sock:/var/run/docker.sock:ro \
    nextcloud/all-in-one:latest
```

2. Next step, I opened the AIO (all-in-one) interface at `short.saihemanth.com:8443`. This webpage gave me the default admin
    password and redirected me to an installation page. It asked for a valid domain name with port 443 open and a valid SSL to
    be used for NextCloud.
    
    ![](/assets/img/nextcloud-post/nextcloud-aio-install.png)

3. The domain was quickly verified and accepted by the tool! I started all the required nextcloud docker containers through
    the UI and waited for the containers to come up. They never did. The apache docker container logs gave me
    this error, `cURL error 35: OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to <domain>:443`.

4. Took a while to figure out, but I realized it's a memory issue. Bitwarden's containers were struggling for resources alongside
    the newly bought up NextCloud containers. NextCloud's Redis container was in particular failing, and the Apache container
    was waiting for the database to be up. So, no NextCloud for me.

5. I resized the droplet to 2vCPUS, 4GB memory, and 50GB Disk and followed from step 1 again. It all worked flawlessly now!

    ![](/assets/img/nextcloud-post/nextcloud-running-containers.png)

6. I quickly set up 2FA for the admin account, and created a different personal user account for daily use.

After 4 hours, I finally have a working NextCloud instance running! *(Not Really)*

## Problem #3: Setting up Backups

NextCloud AIO doesn't seem to have support for automated backups. Admin needs to explicitly generate backups by
clicking a button in settings, upon which an encrypted backup is created and stored. I would've liked
something automated, but I wasn't too eager to explore more.

1. Upon generating backups through the UI, they land as encrypted files in "/root/nextcloud_backup" directory. This works for
   now.
2. I need to sync these backups from the DO droplet to my local backup disk. What better tool than rsync? I rsynced with
    archiving and compressing enabled to hopefully make it faster.
```sh
rsync -avzh root@100.101.102.103:/root/nextcloud_backup /backup-disk/remote_backup/nextcloud_backup
```

The files were downloading at 25KBps! It will take forever to download 15GB backups. This speed was abysmal and barely
viable as a solution. I forboded to one of the reasons before. This server is all the way in Canada, and this
trans-continental data transfer was poor. However, I partially felt it also had to do with DigitalOcean itself.

# Migrating to AWS

## Problem #4: Improve transfer speeds

There were a couple of reasons why I wanted to dabble with AWS,

1. **Cheaper Servers** - AWS VPSs are about 5\\$ cheaper compared to a similar configuration in DO. The exact setup is
    20\\$ on AWS Lightsail, while DO was 24\\$ per month.
2. **Better Integrations** - AWS is known for its wide variety of services and integrations, and I believed they could
    aid me later.
3. **Curiosity** - I probably could do something similar on DigitalOcean, but I wanted to play with AWS and see what
    AWS is all about.

Everything after this was smooth,

1. Started a lightsail instance in Mumbai datacenter with 2GB RAM and 1vCPU. This config's free for 3 months.
2. Setup the network config,
    * Open ports 443, 8080, and 8443 on both TCP and UDP.
    * Create a static IP and attach it to the instance.
    * Add DNS mapping for this static IP on Bluehost.
3. Follow all steps from Problem #2 to install and set up NextCloud AIO.

Once again, I finally have a working NextCloud instance running!

## Problem #5: Stress Testing & Performance

8 hours in.

I downloaded 600MB worth of pictures from Google Photos and tried uploading them to NextCloud. The app
promptly became unresponsive and crashed. I couldn't SSH into the server either. Nice 👌.

![](https://c.tenor.com/mX7EHu4a69AAAAAC/catdestroy-catknockover.gif)

It was clear that 1vCPU was insufficient for this instance, so I bumped it up to 2vCPUs and 4GB RAM. Unlike
DigitalOcean, AWS had no option to resize the existing server itself. I had to create a new larger lightsail instance
using a backup of the previous one.

One other observation I had was that NextCloud does not send previews for gallery views by default. It instead
sends over the entire image! One way to circumvent this is to use the
[previewgenerator](https://github.com/nextcloud/previewgenerator) NextCloud app.
This requires a cron job to continuously generate previews for any recently uploaded media.

```
*/10 * * * * docker exec -it <docker-process-id> ./occ preview:pre-generate
```

I tried out rsync of backups similar to before, and this time it downloads at 8MBps!

Okay, I finally finally have a working NextCloud instance! For real.

# Retrospective

Comparing my final setup against the goals I set initially.

### 🔘 Reliability

I'm not happy with storing my files in the instance's disk. It seems fragile and bound to fail at some point.
Increasing storage also requires me to bump up the instance's config, which is far from ideal. For e.g., the current
lightsail instance comes with 80GB disk. If I need more space, I'll have to increase the VPS's specs including vCPUs.

If the server dies, I have mechanisms to recover the setup using a backup. Creating and syncing backups is a
low effort task. However, I might lose the recent uploads that aren't backed up. There's scope for improvement here.

### ☑️ Security

I don't see any obvious security flaws here. Running [NextCloud Scan](https://scan.nextcloud.com/) gives an A rating.
There's SSL in place. Only required ports are open. There's probably more I can do here, but they only yield
diminishing returns. 

### ☑️ Responsiveness

After I moved to AWS Lightsail, the website and Android app are blazing fast. The uploads happen at upwards of 8MBps,
and the experience is super smooth.

### ❌ Cost Effectiveness

The entire setup is just one Lightsail instance. The lightsail instance (2vCPUs/4GB/80GB Disk)
costs \\$20 per month. In contrast, Google One subscription costs Rs.135 in India, \\$1.7.

**I cannot justify having a feasible replacement if it costs 11 times more!**

![](https://y.yarn.co/3220d811-2f03-4f77-b352-7f31160fa35f_text.gif)

*Stay tuned for Part 2 of this series!* Follow me on [Twitter](https://twitter.com/saihemanth9019) for more :)