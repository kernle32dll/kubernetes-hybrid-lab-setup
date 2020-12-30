# 00: Hard- and software decisions

In this section, I will talk a bit about how the machines will be set up, and
the reasons for each of my decisions.

If you are not interested in that, you can immediately skip to the next section.

## OS

TL; DR; **Both master and worker will be setup with Arch Linux.**

When I started thinking about my setup, I never used Arch Linux before.
I did however hear a lot of good and bad things about it, and it was due
time for me to make my own picture. 

The Raspberry Pi Arch Linux setup was especially interesting, as it posed some
additional - for me unforeseen - challenges in setup.

I briefly considered Alpine Linux, as I am a great fan of it (both bare metal
and as a Docker base image), but it did not really click for me while testing it
out on my Pi. I also had some doubts about getting Kubernetes getting to run
flawlessly.

## File system

TL; DR; **The master will use a plain BTRFS root, the worker a BTRFS RAID1 root.**

I am a fan of btrfs. It took me some time to get into it, but I have been using it
ever since for my desktop and laptops. It was due to get some testing done with
btrfs on a server.

The main advantage with btrfs in this setup is the ability to snapshot, and easily
play around, and recover F-ups.

The worker will get a RAID1 btrfs, as I got my hands on two cheap SAS 12gb/s SSDs.
After some devastating hard drive data losses in past years, I tend to always use
RAID 1 where I can. Remember tho: **RAID is not a backup!**. I will come back to this
much later.

## Encryption

TL; DR; **Yes**

Seriously, I never install any system of any kind nowadays without strong encryption.

It serves two purposes:

- For mobile devices (laptops), I can rest assured that no data can be stolen in case
my device gets stolen. I keep a lot of important stuff around, and I really don't
want these to get into the wrong hands.

- For stationary devices (desktops, servers), its basically the same thing. But I
keep the "theft" thought a bit less literal. You will figure out what I mean by that.
Basically, I don't want anyone to have access to any kind of stored data on my
machines.

I had some doubts if the Raspberry Pi 4 was up to the job, but so far I am
pretty pleased.

## Remote installation

TL ; DR; **We will prepare dedicated installation media, with SSH servers**

I'm not a big fan of installing the system with display and keyboard attached.
Especially if like in the case of the worker server, there is only VGA output.

Nothing beats the comfort of sitting at your desk.

So, we will prepare dedicated installation media, which include an auto-started
SSH server. This will allow to do the installation completely remote. This also
has the advantage of being a good base for installation automation, which I
sadly did not get to.

## Raspberry Pi 4 hardware setup

TL; DR; **Industrial grade micro SD and a decent cooler**

I pretty much left the Pi as is. As it tends to get a bit warm I installed a
[GeeekPi Raspberry Pi Low-Profile CPU](https://www.amazon.de/dp/B07ZV1LLWK/),
and exchanged the fan for a [Noctua NF-A4x10 5V](https://www.amazon.de/dp/B00NEMGCIA/).

For now the Pi sits around loosely in my rack, but I will eventually get
(or 3D print) myself a rack mount to properly mount it. Probably when I get
some more Pis to extend my cluster.

For storage, I got myself some industrial grade micro sd card, which can (should?)
sustain the load the filesystem will suffer from running a Kubernetes cluster.
After reading trough some stuff, I settled on a
[32gb SanDisk MAX ENDURANCE](https://www.amazon.de/dp/B084CJLNM4/) card.

On a side note: Get a decent card reader! I blew **two** ancient USB card readers
while setting up everything. I eventually got a USB 3 based multi-reader.

## Dell R420 hardware setup

TL; DR; **Dual Intel Xeon E5-2430L V2, 32gb RAM, LSI 9302-8I HBA, 2x SAS 12gb/s
SSDs, 3x SAS 12gb/s HDDs**

Lets talk a bit more about the worker setup in detail.

The R420 sports a custom motherboard, with two CPU sockets, and 2x 6 for a total
of 12 RAM slots. I bought the server with two `Intel Xeon E5-2430`, which I
quickly discarded - just like the preinstalled disks. For a few bucks I got
myself two `Intel Xeon E5-2430L V2` instead. Lower TDP, for basically the same
power.

For RAM, I paid a bit of a premium, and got myself 8x 4gb sticks, for a total
of 32gb. I thought its fair as a starting point, and left me with 4 more slots
to populate. 4gb sticks are dirty cheap on ebay.

For the disks and their connectivity, I pretty much went enthusiast level
(considering what the board offers). I dropped the built in Dell H710 Mini
controller, as its only SAS 6gb/s - and I was set early on to use fast SSDs.
I had good experiences with a LSI card in my NAS server, and thus went with
a LSI 9302-8I HBA variant (note: it was dirty cheap, and seemed to be a IT
cross-flashed card. I don't mind, it works great).

For the OS installation, and some fast spare storage (read: DB containers)
I got myself some Lenovo branded HGST SAS 12gb/s SSDs, with 300gb each. As
alluded in the file system section, they will be set up with RAID1 for good
data integrity. The main cluster storage will be seated on another RAID1*,
consisting of three Seagate Exos ~2tb HDDs.

*Note: RAID1 on btrfs is a bit different than you might expect. I did this
intentionally.