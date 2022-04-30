---
title: Building a Home Server - Part 1
date: 2022-04-01 09:00:00 +0530
categories: [Programming]
tags: []     # TAG names should always be lowercase
math: true
mermaid: true
toc: true
published: true
---

# Objectives

As a privacy nut, I'm not comfortable storing GBs of photos, videos, and documents on OneDrive and Google Drive. My most
personal memories locked away in some heartless data warehouse, possibly used for advertising.

I've been toying with the idea of replacing all of my cloud-based applications with something
custom-built. This includes Google Photos, Google Drive, One Drive, and Digital Ocean VMs.

To summarize,
1. A place to save photos, videos, and files reliably and redundantly
2. A web server
3. A dev VM
4. A password manager server

Of course, the main goal is to have fun and learn :)

# Choosing the Hardware

The scale of my requirements is pretty minimal, so I don't need a dedicated server PC. I can't use
a pre-built NAS either as it isn't customizable enough. So, I decided to build
a consumer PC. As a regular LTT viewer, I have basic knowledge of PC building.

## Processor

This minimal server does not really require large core/thread counts or high frequency cores. So, I decided to
settle on the affordable **Intel i5-11400** 2.6GHZ processor. It has 6 cores and 12 threads. This CPU comes
with an integrated graphics card. Also, it cannot be overclocked, which is fine because I don't want
to spend time thinking about efficient cooling.

Amazon Link: [https://www.amazon.in/gp/product/B08X6JPK4K](https://www.amazon.in/gp/product/B08X6JPK4K)

## Memory

Ideally, I should get 32GB of DDR4 ECC RAM. However, it's expensive and hard to get ECC supported RAM, CPU, and
motherboard in India. Non-ECC RAM sticks are generally unlikely to fail, and I'll be relying on backups extensively. I've settled
on using two 8GB **G Skill Ripjaws 3200MHz DDR4** RAM Sticks. I can expand to 32GB in the future.

Amazon Link: [https://www.amazon.in/gp/product/B01J6OA8RU](https://www.amazon.in/gp/product/B01J6OA8RU)

## Graphics Card

A server doesn't need one. The integrated graphics card in the i5 processor should more than do the job.
I considered chucking in a 2060 to play Minecraft when I'm free, but all GPUs are heavily scalped right now :(

## Storage

I need fast storage. So, I'm buying two **Samsung 970 Evo Plus 500GB M.2 SSDs**. One will be used for Host OS and VM Images.
Another will probably host all the required fast-access data. I haven't nailed down the specifics yet. I need to buy
two 5TB HDDs to store backups, images, and videos later.

Amazon Link: [https://www.amazon.in/gp/product/B07MFBLN7K](https://www.amazon.in/gp/product/B07MFBLN7K)

## Motherboard

Choosing this was the hardest. I wanted to maximize for the M.2 slots and SATA ports available. I ended up with
**ASUS TUF Gaming B560M-Plus WiFi mATX Motherboard**. It has two M.2 slots, Six SATA ports and Thunderbolt 4 port.

Amazon Link: [https://www.amazon.in/gp/product/B08CMVNJ58](https://www.amazon.in/gp/product/B08CMVNJ58)

## Power Supply

The CPU I've chosen is a 65W CPU and I don't need a dedicated GPU. So, my power requirements are pretty light. I've decided
to go with a Corsair non-modular 550W PSU. It's cheap and allows for future expansions, in case I decide to pop in an Nvidia 2000
series graphics card.

Amazon Link: [https://www.amazon.in/gp/product/B07YVVNK6Y](https://www.amazon.in/gp/product/B07YVVNK6Y)

## Case

Any mid-tower case that could fit an mATX motherboard was sufficient for me. I went with the popular **Corsair 4000D Airflow** case.
It has two 2.5" SATA SSD mounts and two 3.5" HDD bays. If I ever need more disks, which will not be the case for the next few years,
I'll change the case.

Amazon Link: [https://www.amazon.in/gp/product/B08C7BGV3D](https://www.amazon.in/gp/product/B08C7BGV3D)

# Building the PC

Assembling the PC itself was pretty straightforward. Linus Tech Tips has some great PC Building guides, especially this recent one.

<iframe width="560" height="315" src="https://www.youtube.com/embed/BL4DCEp7blY" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

After getting some thermal paste on my hands, fearing I'll damage the board by pressing on the RAM sticks too hard, struggling
to figure out where to mount the motherboard in case, and struggling to keep the cables tucked in neatly, I managed to get it working
on the first try.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">And... Done. <a href="https://t.co/0oBJXqgZA7">pic.twitter.com/0oBJXqgZA7</a></p>&mdash; Sai Hemanth (@saihemanth9019) <a href="https://twitter.com/saihemanth9019/status/1507681065166864393?ref_src=twsrc%5Etfw">March 26, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
