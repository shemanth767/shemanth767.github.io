---
title: Replacing Google Drive - Part 1
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

![](/assets/img/nextcloud-post/basic-nextcloud-plan.jpg)

# Preparing my existing server

I've been using a Digital Ocean droplet for the past 2 years to host Bitwarden password manager and some ad-hoc
testing for ongoing projects.

![](/assets/img/nextcloud-post/previous-do-server.png)
_Digital Ocean server configuration_

Couple of things to note,
* **TOR1** is a datacenter in Toronto, Canada. It's pretty far network-wise from Hyderabad. However, this wasn't a
    concern for Bitwarden as the mobile app and extension caches all passwords frequently.
* **Memory of 2GB** - The absolute minimum required for installing Bitwarden.
* **50GB Disk** - This was originally only 25GB, because why would a password manager need more?

Okay, first problem to tackle.

## Problem #0: Setting up SSH login to the server
I've recently moved to a new PC, so I had to enable SSH login from this one. It was a very straight-forward process.

1. Copy over public key from local to remote server's /root/.ssh/authorized_keys
2. Restart ssh daemon, `service sshd restart`

## Problem #1: Change the domain name of the server

I've setup an A record on Bluehost, where I bought my domain, to point "verylongname.saihemanth.com" to the static IP
of my server. I wanted to change it to "short.saihemanth.com" instead. Updating it on Bluehost took only 2 mins, and
the DNS record was published in less than 5 mins.

![](/assets/img/nextcloud-post/short-domain-dns-record.png)
_DNS Record for new short domain name_

Bitwarden was being a pain in the ass for switching domain names, more specifically the in-built certbot. It was supposed
to be a single config change in `/opt/bitwarden/bwdata/config.yml` and then run `./bitwarden.sh restart`. This worked, but the
generated SSL certificates were still linked to the old domain.

I thought, "Okay, no problem. Let me just trigger the renew certificate command. It should pick up the new domain. Right?"
Wrong!

![](/assets/img/nextcloud-post/certbot-renew.png)
_Certbot proceeded to use the old URL!_

I proceeded to delete all certificate files used specifically by certbot, in the hopes that it'll try to pick up the correct
domain name from config next time. That still didn't work. I struggled with fixing this for over an hour!

![](https://c.tenor.com/g_H8p8ecUyEAAAAC/leslie-angry.gif)

I exported all my passwords using the UI and uninstalled bitwarden: `./bitwarden.sh uninstall`. I
reinstalled and setup bitwarden and certbot again. Finally, I imported my vault through the previous
export. The whole process took less than 20 mins.

Not elegant, but problem solved!

# NextCloud

## Problem #2: Setting up Nextcloud

I took a backup of the droplet to start with — in case I brick the server. DigitalOcean bills me for keeping backups too.
Thanks DO 👍

1. Installing NextCloud itself was very simple. I just had to follow nextcloud all-in-one's [README](https://github.com/nextcloud/all-in-one).
    What does *all-in-one* mean? I didn't know, I just needed the cloud storage functionality and this was easy to setup.
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

2. Next step, I opened the AIO interface at `short.saihemanth.com:8443`. NextCloud gave me default admin password and redirected
me to the installation page. I had to enter a valid domain name that I was using for NextCloud.  
![](/assets/img/nextcloud-post/nextcloud-aio-install.png)

3. The domain was quickly verified and accepted by the tool! I started all required nextcloud docker containers through the tool.
    I waited for the containers to come up, but they never did even after 20 mins. I quickly opened the docker logs and saw this
    error under the apache logs, `cURL error 35: OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to <domain>:443`.

4. Took a while to figure out, but realized that it's a memory issue. Bitwarden's containers were struggling for resources alongside
    the newly bought up NextCloud containers. Redis container was in particular failing, and the Apache container
    was waiting for the database to be up. So, no NextCloud for me.

5. I resized the droplet to 2vCPUS, 4GB memory and 50GB Disk, and followed from step 1 again. It all worked flawlessly now!

![](/assets/img/nextcloud-post/nextcloud-running-containers.png)

6. I quickly set up 2FA for the admin account, and created a different personal user account for daily use.

After 4 hours, I finally have a working NextCloud instance running!  

## Problem #3: Setting up Nextcloud Backups

NextCloud AIO doesn't seem to have support for automated backups. Backups are taken to a pre-defined directory only upon the user
clicking a "Generate Backup" button. I wasn't too eager to explore more solutions either.

1. Upon generating backups through the UI, they land as encrypted files in "/root/nextcloud_backup" directory. This works for
   now.
2. I need to sync these backups from the DO droplet to my local backup disk. What better tool than rsync? Archiving and compressing
    is enabled in the rsync to hopefully keep it fast.
```sh
rsync -avzh root@100.101.102.103:/root/nextcloud_backup /backup-disk/remote_backup/nextcloud_backup
```

The files were downloading at 25KBps! It will take forever to download 15GB backups. This speed was abysmal and barely
viable as a solution. I forboded to one of the reasons before. This server is all the way in Canada, and this
trans-continental data transfer was poor. However, I partially felt it also had to do with DigitalOcean itself.

# Migrating to AWS

## Problem #4: Improve transfer speeds

There were a couple of reasons why I wanted to try this out using AWS,

1. **Cheaper Servers** - AWS VPS's are 5\\$ cheaper compared to similar configuration in DO. The exact setup is
    20\\$ on AWS Lightsail, compared to the earlier 24\\$ per month.
2. **Better Integrations** - AWS is known for its wide variety of services, and I believed they could aid me later.
3. **Curiosity** - I probably could do something similar on DigitalOcean, but I wanted to play with AWS and see what the
    hype is all about.

Everything after this was largely smooth,

1. Started a lightsail instance in Mumbai datacenter with 2GB RAM and 1vCPU. This config's free for 3 months.
2. Setup the network config,
    * Open ports 443, 8080 and 8443 on both TCP and UDP.
    * Create a static ip for the instance.
    * Add DNS mapping for this static IP on Bluehost.
3. Follow all steps from Problem #2 to install and setup NextCloud AIO.

Once again, I finally have a working NextCloud instance running!

## Problem #5: Stress Testing & Performance

8 hours in.

I downloaded 600MB worth of pictures from Google Photos and tried uploading to NextCloud. The app prompty became
unresponsive and crashed. I couldn't SSH into the server either. Nice 👌.

![](https://c.tenor.com/mX7EHu4a69AAAAAC/catdestroy-catknockover.gif)

It was clear that 1vCPU was not sufficient for this instance, so I bumped it up to 2vCPUs and 4GB RAM. Unlike
DigitalOcean, AWS had no option to resize the existing server itself. I had to create a new larger lightsail instance
using a snapshot of the previous instance.

NextCloud also does not send previews for gallery views by default. It instead sends over the entire image! One way
to circumvent this is to use the [previewgenerator](https://github.com/nextcloud/previewgenerator) NextCloud app.
This requires a cron job to continuously generate previews for any recently uploaded images.

```
*/10 * * * * docker exec -it <docker-process-id> ./occ preview:pre-generate
```

I tried out rsync of backups similar to before, and this time it downloads at 8MBps!

Okay, I finally finally have a working NextCloud instance! For real.

# Retrospective

Comparing my final setup against the goals I set initially.

### ☑️ Encrypted and automated backups

Umm, this is kind of in place. NextCloud AIO does not allow for automated backups, but it's very low effort to generate
and sync a backup. I'm going to consider this as done.

### ☑️ Security

I don't see any obvious security flaws here. Running [NextCloud Scan](https://scan.nextcloud.com/) gives an A rating.
There's SSL in place. Only required ports are open. There's probably more that I can do here, but they only yield
dimnishing returns. 

### ☑️ Responsiveness

After I moved to AWS Lightsail, the webapp and Android app are blazing fast. The uploads happen at upwards of 8MBps,
and the experience is super responsive.

### ❌ Cost Effectiveness

The entire setup is just one Lightsail instance and one static IP. The lightsail instance (2vCPUs/4GB/80GB Disk)
costs \\$20 per month. In contrast, Google One subscription costs Rs.135 in India, \\$1.7.

**I cannot justify having a feasible replacement if it costs 11 times more!**