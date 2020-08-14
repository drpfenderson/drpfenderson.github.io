---
published: true
title: "Network Connection Error on DigitalOcean Ubuntu 20.04 Upgrade Resolved!"
excerpt: "After upgrading a remote server to the newest version of Ubuntu, the network connection disappeared. This guide talks about how I fixed the issue."
category: guides
---

There are a number of services I run for various organizations for work, and keeping them all up to date can be a pain. Even in the best of circumstances, a package dependency or custom configuration can break what would normally be a smooth upgrade. This reality is something I just experienced when trying to upgrade my [DigitalOcean](https://m.do.co/c/193aa52834cd) droplet's version of Ubuntu.

(This post isn't sponsored, and I hate advertising, but I heckin love DigitalOcean. They are my host of choice for many personal and professional operations. If you use [my referral link](https://m.do.co/c/193aa52834cd), you'll get [a $100, 60-day credit](https://m.do.co/c/193aa52834cd) to use their services, as well as helping cover the bill for my own experiments.)
{: .notice}

For most long-term projects that I'm working on, a stable base and infrequent changes are a real boon to reducing headaches. That's why I usually go with Ubuntu's [Long-Term Support (LTS) version](https://wiki.ubuntu.com/LTS), which, aside from [security updates and major bugfixes](https://ubuntu.com/about/release-cycle), only updates every couple years. The machine I was working with was still running 18.04, but the newest 20.04 LTS has [been out for a few months](https://ubuntu.com/blog/whats-new-in-ubuntu-desktop-20-04-lts) and just received [its first point release](https://www.zdnet.com/article/ubuntu-linux-20-04s-first-point-release-arrives/). This is the first round of major bugfixes and updates, and often signals that the new version is relatively safe to upgrade.

After a few false-starts during the upgrade process, the rest of the install seemed to go smoothly. Until the system needed to reboot at the end. Suddenly I couldn't reach the machine anymore. Trying to access the remote box using SSH returned a connection error. Viewing the machine from my control panel showed that it was up and running, and I could access it using the DigitalOcean terminal, so I knew it was mostly okay.

Time to investigate. Pinging a random url shows that I have not network connection.

* Trying to  `sudo ip link set eth0 up` returned `Cannot find device 'eth0'.
* Running `ifconfig -a` returns an interface `ens3` and `lo`. Okay, weird, so there's the problem. There should be an `eth0` interface, so....where did it go?
* Searching through logs with `dmesg | grep -i eth`, I see `ens3: renamed from eth0`.

After some fumbling around on my own, I took to the web and found [a forum post by a user](https://www.digitalocean.com/community/questions/no-internet-connection-after-droplet-reboot) who was experiencing a similar issue. Apparently when I upgraded, it removed a couple of important packages, like `cloud-init` and `landscape-common`.

* `cloud-init` is a package that helps the instance pull configuration information from the cloud host, DigitalOcean in this case.
* `landscape-common` is a similar program that contains libraries for deploying, monitoring, and managing Ubuntu servers at scale.

This leaves us with 2 tasks. First, we need to get the network interface up and running so we can actually install software and connect over SSH. Second, we need to install the missing packages so that our network works after reboot. In order to connect to the server, I had to access the console for the droplet from my control panel. This allows me to bypass SSH and directly control the machine, even when it is having network issues.

These are the steps I followed to fix my issue:

1. Pulled the MAC address for the network interface with `tail /etc/udev/rules.d/70-persistent-net.rules`.
2. I created a rules file for the network interface with `sudo touch /etc/udev/rules.d/70-persistent-net.rules`. Then, using `nano /etc/udev/rules.d/70-persistent-net.rules`, I filled the document with the following
```
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="INSERT MAC ADDRESS HERE", NAME="eth0"
```
Make sure to replace the value for `INSERT MAC ADDRESS HERE`, leaving the quotes on either side intact.

3. Reboot with `sudo reboot`, and make sure that interface is still available with `ifconfig -a`.
4. Hopped over to my DigitalOcean control panel, to the **Networking** section of the droplet in question. Made note of the IP address, public gateway, and subnet mask.
5. Ran the following command to bind the interface to the correct network information, substituting `IPADDRESS` for my IP address, and `SUBNET` with the subnet value, both noted from the previous step.
```
sudo ifconfig eth0 IPADDRESS netmask SUBNET up
```
6. Added the gateway address with the following command, substituting `GATEWAY` with the gateway address noted above.
```
sudo ip route add default via GATEWAY
```
At this point, I was able to access my box using SSH, but still decided to use the console for stability. Also, the changes I made were not persistant, and remote servers were still not resolving, so there was still work to do.
7. Changed my DNS nameserver by editing the configuration file:
```
sudo nano /etc/resolv.conf
```
and changed the `nameserver` value to 8.8.8.8, which is [Google's free DNS service](https://developers.google.com/speed/public-dns/docs/using).
At this point, I was finally able to access the internet. `ping duckduckgo.com` was returning packets, but we still weren't to the point of settings sticking.
8. Installed the missing packages, first making sure that I wasn't missing any updated system packages.
```
sudo apt update
sudo apt upgrade
sudo apt install cloud-init
sudo apt install landscape-common
```

And that was it! I was able to reboot, and see that it pulled new configuration options from DigitalOcean, including the original, correct DNS settings. Testing SSH showed that everything was in working order, and I was able to continue with my tasks. Now I leave this here with the hope that someone in the future has to search around less than I did.