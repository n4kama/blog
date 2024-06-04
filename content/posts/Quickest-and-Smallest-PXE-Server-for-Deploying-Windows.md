---
title: "Quickest and Smallest PXE Server for Deploying Windows"
date: 2024-06-04
# weight: 1
# aliases: ["/first"]
tags: ["powershell", "microsoft", "windows", "osdcloud", "infrastructure", "winpe", "wds", "sysadmin", "pxe", "ipxe", "alpine", "linux", "dhcp", "tftp", "http", "network", "linux"]
author: "Nakama"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: "Setting up a PXE infrastructure to deploy Windows workstations with OSDCloud."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: false
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/n4kama/blog/blob/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

On my previous post about [Using Powershell and OSDCloud to automate Windows workstation deployment](https://blog.baguet.org/posts/quit-using-microsoft-deployment-toolkit/), I mentionned a second part on :  
**Setting up a PXE infrastructure to deploy Windows workstations with OSDCloud.**  
So there we are...

At the time, I naively believed I could deploy a functional iPXE infrastructure for BIOS, UEFI, and especially **UEFI with secure boot**. What a mistake that was...

> Quick explaination : Secure Boot ensures that the boot chain is signed by a PC manufacturer. Therefor, it gets very difficult to boot from a custom made PXE/iPXE infrastructure. There are ways, one of which would be to [pay a lot of money to get your boot chain to be signed](https://ipxe.org/appnote/etoken). Another way would be to pay less money for an already signed PXE server. 
> I'm not going to cover these 2 options here, but keep in mind that they do exist.

## I want to have it ALL : BIOS, UEFI and UEFI **with** Secure Boot

Sadly (or not), the simplest and cheapest (it's free) option is to use a Windows server with WDS (Windows Deployment Services). Honestly, it works pretty well and can be configured quickly.

I know my previous post aimed at convincing you to abandon Microsoft Deployment Toolkit and I still stand by it. Using OSDCloud as a replacement for building a boot image and then using WDS to deploy it is optimal in my opinion.

## I don't care about Secure Boot, give me the quickest and smallest PXE server (BIOS and UEFI compatible)

We are going to setup an alpine server with :
- DHCP server
- TFTP server
- HTTP server
- iPXE

### Install Alpine server

1. Boot up a server with alpine standard iso and login with `root`.
2. Run the wonderful `setup-alpine` installer and follow the instructions :
   1. Select keyboard.
   2. Select hostname, in my case it's `rhea`.
   3. Configure your network interfaces, in my case `eth0` and `eth1`. One of them is on a network open to the Internet, the other on an isolated network dedicated to workstation deployment.
      1. For `eth0` : I assume you have basic network knowledge. You will *probably* have to set this up manually according to your own network specifications.
      2. For `eth1` : You are free to select your prefered subnet as it will be NATed later on. I'll go with `10.0.0.1` for the ip address and gateway, `255.255.255.0` as netmask (253 free addresses is enough in my case).
   4. Configure the password, timezone, apk mirror.
   5. No new user appart from `root` in my case.
   6. Configure `openssh` for remote access.
   7. Configure disks ( I use `sys` mode).
   8. Ensure DNS resolution is working with a simple `ping google.com`. Use `setup-dns` to fix any issue if needed.
   9. `poweroff` the machine and remove the installation media.

You should probably do the good ol' `ssh-copy-id` followed by `PermitRootLogin prohibit-password` (`/etc/ssh/sshd_config`) shenanigans 👀.  
Run `rc-service sshd restart` afterwards.

You may now ssh into `rhea` 🧙‍♂️.

### Setup NAT

```shell
# Enable IP forwarding (necessary for routing between networks)
echo "net.ipv4.ip_forward=1" | tee -a /etc/sysctl.conf
sysctl -p

# Enable NAT
apk add iptables
rc-update add iptables
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE    # eth0 is the net card of my open network 
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT           # eth1 is the net card of my isolated network
/etc/init.d/iptables save
```

> I am using `MASQUERADE` as I don't have a fixed external IP address on eth0. If you do (and you probably will), consider using SNAT instead for better performance (it wouldn't really change much in our use case).

### Setup DHCP server

Install the DHCP server:
```shell
apk add dhcp
```

Edit `/etc/dhcp/dhcpd.conf`:
```dns
subnet 10.0.0.0 netmask 255.255.255.0 {
    range 10.0.0.100 10.0.0.200;        # Use your own subnet
    option routers 10.0.0.1;            # In my case, IP of eth1 interface
    option domain-name-servers 8.8.8.8; # Use your prefered DNS server
    next-server 10.0.0.1;               # TFTP server IP
    option bootfile-name "ipxe.efi";
}
```

Configure the DHCP server to start at boot and manually start it now:
```shell
rc-update add dhcpd
rc-service dhcpd start
```

> 💡Tip: Check IP address leases in `/var/log/messages`

### Setup TFTP Server

```shell
apk add tftp-hpa
rc-update add in.tftpd
rc-service in.tftpd start
```

> Our PXE files will be copied in `/var/tftpboot/` later

### Setup HTTP Server

```shell
apk add lighttpd
rc-update add lighttpd default
rc-service lighttpd restart
```

> Our boot image files will be copied in `/var/www/localhost/htdocs/` later.

### Setup iPXE

We now have a functional server with all the services we need. We are, however, missing a vital part:
- A bootable image 
- A PXE for the machine to boot on

Let's handle that second part first.

```shell
apk add git make binutils mtools perl xz-dev libc-dev gcc
cd /srv
git clone https://github.com/ipxe/ipxe.git
cd ipxe/src
```

We will now create `netboot.ipxe`:
```
#!ipxe

# Automatically configure interfaces
dhcp

# Network info
echo next-server is ${next-server}
echo bootfile is ${filename}
echo MAC address is ${net0/mac}
echo IP address is ${ip}

# Set HTTP server address
set server http://${next-server}
echo server is ${server}

# Set architecture
cpuid --ext 29 && set arch x86_64 || set arch i386
echo arch is ${arch}

# Load boot loader for Windows
echo Loading Wimboot
kernel ${server}/wimboot
initrd ${server}/BCD        BCD
initrd ${server}/boot.sdi   boot.sdi
initrd ${server}/boot.wim   boot.wim
boot
```

We now need to compile our iPXE with our netboot configuration. Lets create `build.sh` and `chmod +x build.sh` it:
```shell
make bin-x86_64-efi/ipxe.efi EMBED=netboot.ipxe
make bin-x86_64-efi/snponly.efi EMBED=netboot.ipxe
cp bin-x86_64-efi/ipxe.efi /var/tftpboot/
cp bin-x86_64-efi/snponly.efi /var/tftpboot/
```

> - `ipxe.efi` is the main artifact that contains all UEFI drivers, and is the most commonly used.
> - `snponly.efi` can also be used for chainloading boot, if ipxe.efi is not working, which is the case for when I ran tests in VirtualBox for instance. It uses SNP (Simple Network Protocol) or NII (Network Interface Identifier Protocol) instead of UEFI.
> - You could also build `snp.efi` or `undionly.kpxe` (BIOS) if needed.

Download [`wimboot`](https://ipxe.org/wimboot) from iPXE's github with:  
```shell
wget https://github.com/ipxe/wimboot/releases/latest/download/wimboot -P /var/www/localhost/htdocs/
```

Now we will need `BCD`, `boot.sdi` and `boot.wim`. For that we will use our OSDCloud ISO from my previous post (remember this is part 2 👀).

From my host, I mounted my OSDCloud ISO and ran in Powershell:
```powershell
scp D:\Boot\BCD rhea:/var/www/localhost/htdocs/BCD
scp D:\Boot\boot.sdi rhea:/var/www/localhost/htdocs/boot.sdi
scp D:\sources\boot.wim rhea:/var/www/localhost/htdocs/boot.wim
```

## Conclusion

Setting up a PXE server yourself is very educational and a great way to understand the workings of this technology.

However, given that all new PCs are now delivered with Secure Boot enabled, it's probably simpler and more enjoyable to use WDS in combination with OSDCloud for deployment.
