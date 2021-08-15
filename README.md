This guide demonstrates how to build a wired/wireless router using the PC Engines [APU platform](https://www.pcengines.ch/apu.htm) and a free operating system like [OpenBSD](https://www.openbsd.org/) or [Debian](https://www.debian.org/releases/jessie/amd64/ch01s03.html.en) to be used for [network address translation](https://computer.howstuffworks.com/nat.htm), as a stateful firewall, to filter Web traffic, and more.

I am **not** responsible for anything you do by following any part of this guide!

See also [drduh/Debian-Privacy-Server-Guide](https://github.com/drduh/Debian-Privacy-Server-Guide).

# Overview

The completed router configuration will enable:

* An egress Ethernet interface for Internet routing - can be connected to WAN or a cable modem
* A local wireless interface on `192.168.1.0/24`
* A local Ethernet interface on `172.16.1.0/24`
* A local Ethernet interface on `10.8.1.0/24`
* An additional (4th) Ethernet interface is available on APU4

## Hardware

Order directly from [PC Engines](https://www.pcengines.ch/order.htm) or through a reseller.

This guide should work on any PC Engines APU model. Here is a suggested parts list:

| Part | Description | Cost
|------|-------------|------
| [apu4c4](https://pcengines.ch/apu4c4.htm) | apu4c4 system board | $117.50
| [case1d2bluu](https://pcengines.ch/case1d2bluu.htm) | Enclosure 3 LAN, blue | $9.40
| [ac12vus2](https://pcengines.ch/ac12vus2.htm) | AC adapter 12V 2A US plug | $4.10
| [msata16g](https://pcengines.ch/msata16g.htm) | SSD M-Sata 16GB MLC, Phison S11 | $15.50
| [wle200nx](https://pcengines.ch/wle200nx.htm) | Compex WLE200NX miniPCI express card | $19.00
| 2 x [pigsma](https://pcengines.ch/pigsma.htm) | Cable I-PEX -> reverse SMA | $2.70
| 2 x [antsmadb](https://pcengines.ch/antsmadb.htm) | Antenna reverse SMA dual band | $4.10

To connect over serial, you will need a [USB to Serial (9-Pin) Converter Cable](https://www.amazon.com/gp/product/B00IDSM6BW) and [Modem Serial RS232 Cable](https://www.amazon.com/gp/product/B000067SCH), also available from [PC Engines](https://www.pcengines.ch/usbcom1a.htm).

See [Issue #1](https://github.com/drduh/PC-Engines-APU-Router-Guide/issues/1) for a list of alternative parts.

## Assembly

Clear an area to work and unpack all the materials. Follow the [apu cooling assembly instructions](https://www.pcengines.ch/apucool.htm) to install the heat conduction plate.

Attach the mSATA disk and miniPCI wireless adapter in their respective slots.

See the relevant APU series manual for detailed board information:

* [APU2](https://www.pcengines.ch/pdf/apu2.pdf)
* [APU3](https://www.pcengines.ch/pdf/apu3.pdf)
* [APU4](https://www.pcengines.ch/pdf/apu4.pdf)

**Note** Wireless radio cards are ESD sensitive, especially the RF switch and the power amplifier. To avoid damage by electrostatic discharge, the following installation procedure is [recommended](https://www.pcengines.ch/wle200nx.htm):

1. Touch your hands and the bag containing the radio card to a ground point on the router board (for example one of the mounting holes). This will equalize the potential of radio card and router board.
1. Install the radio card in the miniPCI express socket.
1. Install the pigtail cable in the cut-out of the enclosure. This will ground the pigtail to the enclosure.
1. Touch the I-PEX connector of the pigtail to the mounting hole to discharge, then plug onto the radio card.

To avoid arcing, plug in the DC jack first, then plug the power adapter into mains.

Press `F10` during boot and select `Payload [memtest]` to complete at least one pass.

# Connect over serial

The APU serial connection uses 115200 baud rate, 8N1 (8 data bits, no parity, 1 stop bit).

On OpenBSD, use [cu](https://man.openbsd.org/cu):

```console
$ doas cu -r -s 115200 -l cuaU0
```

On Linux, use [screen](https://www.gnu.org/software/screen/manual/screen.html):

```console
$ screen /dev/ttyUSB0 115200 8N1
```

Or use [minicom](https://linux.die.net/man/1/minicom):

```console
$ sudo minicom -D /dev/ttyUSB0
```

Power on the APU and make note of the firmware version displayed briefly during boot.

# Updating firmware

**Important** Recent firmware versions require [disabling IOMMU](https://github.com/pcengines/coreboot/issues/206#issuecomment-436710629) to work properly with WLE200NX wireless cards.

Check for the latest firmware version at [pcengines.github.io](https://pcengines.github.io/)

To update firmware, first download and extract [TinyCore Linux](https://pcengines.ch/file/apu2-tinycore6.4.img.gz).

Download and import the [firmware signing key](https://github.com/3mdeb/3mdeb-secpack/tree/master/customer-keys/pcengines/release-keys), then check the file signature:

```console
$ curl -LO https://github.com/3mdeb/3mdeb-secpack/raw/master/customer-keys/pcengines/release-keys/pcengines-open-source-firmware-release-4.14-key.asc

$ gpg --import pcengines-open-source-firmware-release-4.14-key.asc

$ gpg apu4_v4.14.0.3.SHA256.sig
gpg: Signature made Fri Aug 13 08:55:30 2021 PDT
gpg:                using RSA key 4825F0B02609904A9F9896BAC06ADEE757C2E791
gpg: Good signature from "PC Engines Open Source Firmware Release 4.14 Signing Key" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 4825 F0B0 2609 904A 9F98  96BA C06A DEE7 57C2 E791

$ shasum -a 256 apu4_v4.14.0.3.rom 2>/dev/null | grep -q $(cat apu4_v4.14.0.3.SHA256 | awk '{print $1}') && echo ok
ok
```

Mount a USB disk and write the TinyCore image, copy the `.rom` file:

```console
$ curl -O https://pcengines.ch/file/apu2-tinycore6.4.img.gz

$ gzip -d apu2-tinycore6.4.img.gz

$ sha256sum apu2-tinycore6.4.img
f5a20eeb01dfea438836e48cb15a18c5780194fed6bf21564fc7c894a1ac06d7  apu2-tinycore6.4.img

$ sudo dd if=apu2-tinycore6.4.img of=/dev/sdd bs=1M
511+0 records in
511+0 records out
535822336 bytes (536 MB, 511 MiB) copied, 22.9026 s, 23.4 MB/s

$ sudo mkdir /mnt/usb

$ sudo mount /dev/sdd1 /mnt/usb

$ sudo cp -v apu4_*.rom /mnt/usb

$ sudo umount /mnt/usb
```

Connect the USB disk to the APU, press `F10` at boot and select the USB disk:

```console
SeaBIOS (version rel-1.14.0.1-0-g8610266a)

Press F10 key now for boot menu

Select boot device:

1. USB MSC Drive Samsung Flash Drive DUO 1100
2. AHCI/0: SB2 ATA-11 Hard-Disk (111 GiBytes)
3. Payload [setup]
4. Payload [memtest]
```

Check the current version:

```console
root@pcengines:~# dmesg | grep apu
[    0.000000] DMI: PC Engines apu4/apu4, BIOS v4.10.0.1 09/10/2019
```

Save the existing version and write the new one:

```console
root@pcengines:~# cd /media/SYSLINUX

root@pcengines:/media/SYSLINUX# flashrom -p internal -r apu4.rom.$(dmidecode -s baseboard-serial-number|tail -n1).$(date +%F)
[...]
Found Winbond flash chip "W25Q64.V" (8192 kB, SPI) mapped at physical address 0xff800000.
Reading flash... done.

root@pcengines:/media/SYSLINUX# flashrom -p internal -w apu4_v4.14.0.3.rom
[...]
Found Winbond flash chip "W25Q64.V" (8192 kB, SPI) mapped at physical address 0xff800000.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
```

Unplug the USB disk and `reboot`

On reboot, select `F10` and `Payload [setup]`. Press `w` to enable BIOS write protection then `s` to save and reboot.

Verify the version by checking serial output during boot:

```
PC Engines apu4
coreboot build 20212705
BIOS version v4.14.0.1
```

From OpenBSD:

```console
$ dmesg | grep bios
bios0 at mainbus0: SMBIOS rev. 2.8 @ 0xcfe8b020 (13 entries)
bios0: vendor coreboot version "v4.12.0.2" date 06/28/2020
bios0: PC Engines apu4
acpi0 at bios0: ACPI 6.0
```

From Debian:

```console
$ sudo dmesg | grep apu
[    0.000000] DMI: PC Engines apu4/apu4, BIOS v4.14.0.1 05/27/2021
```

**Note** APU firmware can also be updated from Debian, without rebooting to TinyCore Linux:

```console
$ sudo apt install flashrom

$ wget https://3mdeb.com/open-source-firmware/pcengines/apu4/apu4_v4.14.0.3.rom

$ sudo flashrom -p internal -w apu2_v4.14.0.3.rom
```

To complete the update, shut down Debian and power off the APU fully, then reboot.

# Prepare OS installer

Use another computer to prepare an installer for either OpenBSD or Debian.

## OpenBSD

Download the installation image - [`amd64/install69.img`](https://cdn.openbsd.org/pub/OpenBSD/6.9/amd64/install69.img) - as well as [`SHA256`](https://cdn.openbsd.org/pub/OpenBSD/6.9/amd64/SHA256) and [`SHA256.sig`](https://cdn.openbsd.org/pub/OpenBSD/6.9/amd64/SHA256.sig) files.

Verify the signatures file and hash of the installation image:

```console
$ cat /etc/signify/openbsd-69-base.pub
untrusted comment: openbsd 6.9 base public key
RWQQsAemppS46LT4dNnAtVUZt51ResyNU35n4OH9yl/r7JcR3B75fO4V

$ signify -C -p /etc/signify/openbsd-69-base.pub -x SHA256.sig install69.img
Signature Verified
install69.img: OK
```

Insert a USB disk. Run `dmesg` to identify its label. Then copy the installation file to the USB disk:

On OpenBSD:

```console
$ doas dd if=install69.img of=/dev/rsd2c bs=1m
```

On Linux:

```console
$ sudo dd if=install69.img of=/dev/sdd bs=1M
```

## Debian

Download the network installation image - [`debian-11.0.0-amd64-netinst.iso`](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/) - as well as [`SHA512SUMS`](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/SHA512SUMS) and [`SHA512SUMS.sign`](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/SHA512SUMS.sign) files.

Verify the signatures file and hash of the installation image:

```console
$ gpg SHA512SUMS.sign
gpg: Signature made Sat Aug 14 13:22:04 2021 PDT
gpg:                using RSA key DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Can't check signature: No public key

$ gpg --keyserver hkps://keyserver.ubuntu.com:443 --recv DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: key 0xDA87E80D6294BE9B: 63 signatures not checked due to missing keys
gpg: key 0xDA87E80D6294BE9B: public key "Debian CD signing key <debian-cd@lists.debian.org>" imported
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   3  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 3u
gpg: Total number processed: 1
gpg:               imported: 1

$ gpg SHA512SUMS.sign
gpg: Signature made Sat Aug 14 13:22:04 2021 PDT
gpg:                using RSA key DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Good signature from "Debian CD signing key <debian-cd@lists.debian.org>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: DF9B 9C49 EAA9 2984 3258  9D76 DA87 E80D 6294 BE9B
```

OpenBSD:

```console
$ grep $(sha512 -q debian-11.0.0-amd64-netinst.iso) SHA512SUMS
5f6aed67b159d7ccc1a90df33cc8a314aa278728a6f50707ebf10c02e46664e383ca5fa19163b0a1c6a4cb77a39587881584b00b45f512b4a470f1138eaa1801 debian-11.0.0-amd64-netinst.iso
```

Linux:

```console
$ grep $(sha512sum debian-11.0.0-amd64-netinst.iso) SHA512SUMS
SHA512SUMS:5f6aed67b159d7ccc1a90df33cc8a314aa278728a6f50707ebf10c02e46664e383ca5fa19163b0a1c6a4cb77a39587881584b00b45f512b4a470f1138eaa1801  debian-11.0.0-amd64-netinst.iso
```

Insert a USB disk. Run `dmesg` to identify its label. Then copy the installation file to the USB disk.

OpenBSD:

```console
$ doas dd if=debian-11.0.0-amd64-netinst.iso of=/dev/rsd2c bs=1m
```

Linux:

```console
$ sudo dd if=debian-11.0.0-amd64-netinst.iso of=/dev/sdd bs=1M
```

Unplug the USB disk and plug it into the APU.

# Installing the OS

Press `F10` at boot and select the USB disk.

## OpenBSD

Set the serial console parameters:

```console
Booting from Hard Disk...
Using drive 0, partition 3.
Loading......
probing: pc0 com0 com1 mem[639K 3325M 752M a20=on]
disk: hd0+ hd1+
>> OpenBSD/amd64 BOOT 3.47
boot> stty com0 115200
boot> set tty com0
switching console to com>> OpenBSD/amd64 BOOT 3.47
boot> [Press Enter]
```

Select the Install option:

```console
Welcome to the OpenBSD/amd64 6.7 installation program.
(I)nstall, (U)pgrade, (A)utoinstall or (S)hell? I
```

When presented with a list of network interfaces, `em0` or `re0` is the Ethernet port closest to the serial port:

```console
Available network interfaces are: em0 em1 em2 em3 vlan0.

Available network interfaces are: re0 re1 re2 athn0 vlan0.
```

Use DHCP or configure a static route:

```console
Which network interface do you wish to configure? (or 'done') [re0]
IPv4 address for re0? (or 'dhcp' or 'none') [dhcp] 192.168.1.2
Netmask for re0? [255.255.255.0]
IPv6 address for re0? (or 'autoconf' or 'none') [none]
Available network interfaces are: re0 re1 re2 athn0 vlan0.
Which network interface do you wish to configure? (or 'done') [done]
Default IPv4 route? (IPv4 address or none) 192.168.1.1
add net default: gateway 192.168.1.1
DNS domain name? (e.g. 'example.com') [my.domain] local
DNS nameservers? (IP address list or 'none') [none] 192.168.1.1
```

Configure the root password and set up a user account:

```console
Password for root account? (will not echo)
Password for root account? (again)
Start sshd(8) by default? [yes]
Change the default console to com0? [yes]
Available speeds are: 9600 19200 38400 57600 115200.
Which speed should com0 use? (or 'done') [115200]
Setup a user? (enter a lower-case loginname, or 'no') [no] sysadm
Full name for user sysadm? [sysadm]
Password for user sysadm? (will not echo)
Password for user sysadm? (again)
```

Select the internal mSATA disk and configure the partions:

```console
Available disks are: sd0 sd1 sd2.
Which disk is the root disk? ('?' for details) [sd0] ?
sd0: ATA, SB2, SBFM naa.0000000000000000 (119.2G)
sd1: Samsung, Flash Drive DUO, 1100 serial.0000000000000000 (29.9G)
sd2: Multiple, Card Reader, 1.00 serial.0000000000000000
Available disks are: sd0 sd1 sd2.
Which disk is the root disk? ('?' for details) [sd0]
No valid MBR or GPT.
Use (W)hole disk MBR, whole disk (G)PT or (E)dit? [whole] W
Setting OpenBSD MBR partition to whole sd0...done.
The auto-allocated layout for sd0 is:
#                size           offset  fstype [fsize bsize   cpg]
  a:             1.0G               64  4.2BSD   2048 16384     1 # /
  b:             4.2G          2097216    swap
  c:           119.2G                0  unused
  d:             4.0G         10941728  4.2BSD   2048 16384     1 # /tmp
  e:            11.9G         19330336  4.2BSD   2048 16384     1 # /var
  f:             6.0G         44359392  4.2BSD   2048 16384     1 # /usr
  g:             1.0G         56942304  4.2BSD   2048 16384     1 # /usr/X11R6
  h:            17.3G         59039456  4.2BSD   2048 16384     1 # /usr/local
  i:             2.0G         95334496  4.2BSD   2048 16384     1 # /usr/src
  j:             6.0G         99528800  4.2BSD   2048 16384     1 # /usr/obj
  k:            65.8G        112111712  4.2BSD   2048 16384     1 # /home
```

**Note** The 111.8G "unused" space partition (`/dev/sd0c`) is actually the [entire disk](https://www.openbsd.org/faq/faq14.html#intro).

Select "Auto layout" to continue, then Enter to finish disk setup.

```console
Use (A)uto layout, (E)dit auto layout, or create (C)ustom layout? [a] A
[...]
Available disks are: sd1 sd2.
Which disk do you wish to initialize? (or 'done') [done]
```

Select a [mirror](https://www.openbsd.org/ftp.html) and start the installation:

```console
HTTP Server? (hostname, list#, 'done' or '?') cdn.openbsd.org
Server directory? [pub/OpenBSD/6.7/amd64]

Select sets by entering a set name, a file name pattern or 'all'. De-select
sets by prepending a '-', e.g.: '-game*'. Selected sets are labelled '[X]'.
    [X] bsd           [X] base67.tgz    [X] game67.tgz    [X] xfont67.tgz
    [X] bsd.mp        [X] comp67.tgz    [X] xbase67.tgz   [X] xserv67.tgz
    [X] bsd.rd        [X] man67.tgz     [X] xshare67.tgz
Set name(s)? (or 'abort' or 'done') [done]
```

After installation is complete, unplug the USB disk and reboot. See the OpenBSD [FAQ](https://www.openbsd.org/faq/faq4.html#Install) for more information.

## Debian

At the install menu, select `Tab` to edit boot options and replace `quiet` with:

```
console=ttyS0,115200n8
```

Select `Enter` and select an available resolution:

```
Undefined video mode number: 314
Press <ENTER> to see video modes available, <SPACE> to continue, or wait 30 sec
Mode: Resolution:  Type:
0 F00   80x25      CGA/MDA/HGC
Enter a video mode or "scan" to scan for additional modes: 0
```

Configure a network adapter - `enp1s0` is the interface closest to the serial port.

Select `Guided - use entire disk and set up LVM` as the partition method. Be sure to select internal mSATA drive and not the USB disk as the installation target (usually `sda`).

Select `Separate /home, /var, and /tmp partitions` as the [partitioning scheme](https://www.debian.org/releases/stable/armel/apcs03.html.en).

During `Software selection` - de-select everything except *SSH server*.

Select the internal mSATA drive and not the USB disk as the GRUB loader target.

# First boot

## OpenBSD

The following boot parameters have been appended to `/etc/boot.conf` by the installer and everything should just work:

```
stty com0 115200
set tty com0
```

## Debian

After the GRUB menu, output may get stuck at:

```
Loading Linux 4.9.0-11-amd64 ...
Loading initial ramdisk ...
```

If so, reboot and press `e` at the GRUB menu to enter edit mode, scroll down and replace the word `quiet` with:

```
console=ttyS0,115200n8
```
    
**Note** If arrow keys do not work in GRUB, try using Emacs key bindings to navigate the text field:

* `Control-B` to move left
* `Control-F` to move right
* `Control-P` to move up
* `Control-N` to move down

Press `Control-X` to continue booting and you should see console output.

**Note** If you get an error like, `Alert! /dev/sdX1 does not exist dropping to shell` and are dropped to an initramfs prompt, reboot and edit the `quiet` line to point to `/dev/sda1` or correct partition.

# First login

## OpenBSD

Log in as `root` and install any [pending updates](https://man.openbsd.org/syspatch), unless already [following -current](https://www.openbsd.org/faq/current.html):

```console
# syspatch
```

Install any pending [firmware updates](https://man.openbsd.org/fw_update):

```console
# fw_update
```

Edit `/etc/doas.conf` to allow the regular user to run [privileged commands](https://man.openbsd.org/doas.conf) without a password:

```
permit nopass keepenv :wheel
permit nopass keepenv root
```

Install any needed software:

```console
# pkg_add vim zsh curl free pftop vnstat
```

Log out as `root` and reboot when finished.

## Debian

Log in as `root` to get started.

If necessary, update GRUB by editing `/etc/default/grub` and removing or replacing `quiet` with `console=ttyS0,115200n8` then update the configuration:

```console
root@pcengines:~# update-grub2
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.9.0-11-amd64
Found initrd image: /boot/initrd.img-4.9.0-11-amd64
```

Install any pending updates or install software:

```console
root@pcengines:~# apt update && apt -y upgrade

root@pcengines:~# apt -y install \
    sudo ssh tmux lsof vim zsh git \
    dnsmasq privoxy hostapd htop lshw \
    curl dnsutils ntp net-tools tcpdump whois \
    make autoconf gcc gnupg apt-transport-https
```

**Optional** Change the default login shell to Zsh for the primary user:

```
root@pcengines:~# chsh -s /usr/bin/zsh sysadm
```

# Configure network interfaces

Ethernet or wireless network interfaces can now be configured.

## OpenBSD

On the APU, set a local network interface address and make it permanent:

```console
$ doas ifconfig em1 10.8.1.1 255.255.255.0

$ echo "inet 10.8.1.1 255.255.255.0" | doas tee /etc/hostname.em1
```

Configure an OpenBSD client with DHCP by following the [Networking FAQ](https://www.openbsd.org/faq/faq6.html) or using a static address:

```console
$ doas ifconfig em1 10.8.1.4 255.255.255.0

$ ping -c 1 10.8.1.1
PING 10.8.1.1 (10.8.1.1): 56 data bytes
64 bytes from 10.8.1.1: icmp_seq=0 ttl=255 time=0.845 ms
```

**Optional** Randomize MAC addresses on boot:

```console
$ echo "lladdr random" | doas tee -a /etc/hostname.em0 /etc/hostname.em1 /etc/hostname.em2
```

## Debian

On the APU and on another computer, determine the interface names available:

```console
root@pcengines:~# lshw -C network | grep "logical name"
       logical name: enp1s0
       logical name: enp2s0
       logical name: enp3s0
       logical name: wlp5s0

user@localhost$ sudo lshw -C network | grep "logical name"
       logical name: eno1
       logical name: enp3s0
```

On the APU, edit `/etc/network/interfaces` to append:

```
auto enp2s0
iface enp2s0 inet static
address 10.8.1.1
netmask 255.255.255.0
gateway 10.8.1.1
```

Where `enp2s0` is the network interface one port away from the serial port.

Restart networking and bring up the interface:

```console
root@pcengines:~# service networking restart

root@pcengines:~# ifup enp2s0
```

On another Linux computer, edit `/etc/network/interfaces` to append:

```
auto eno1
iface eno1 inet static
address 10.8.1.2
netmask 255.255.255.0
gateway 10.8.1.1
```

Then also restart networking and bring up the interface:

```console
$ sudo service networking restart

$ sudo ifup eno1
```

Or on another OpenBSD computer, edit `/etc/hostname.em0` to append:

```
inet 10.8.1.4 255.255.255.0
```

It should now be possible to ping the router:

```console
$ ping -c 1 10.8.1.1
PING 10.8.1.1 (10.8.1.1): 56 data bytes
64 bytes from 10.8.1.1: icmp_seq=0 ttl=64 time=0.519 ms
```

To configure the wireless interface, edit `/etc/network/interfaces` on the APU to include:

```
auto wlp5s0
iface wlp5s0 inet static
address 192.168.1.1
netmask 255.255.255.0
hostapd /etc/hostapd.conf
```

Log out as `root` or reboot to continue.

# Configure SSH

From a client, an SSH connection to the APU should be possible, but not yet authorized:

```console
$ ssh sysadm@10.8.1.1
The authenticity of host '10.8.1.1 (10.8.1.1)' can't be established.
ECDSA key fingerprint is SHA256:AAAAA.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.8.1.1' (ECDSA) to the list of known hosts.
Permission denied (publickey,password).
```

If using a [YubiKey](https://github.com/drduh/YubiKey-Guide), copy its public key to clipboard:

```console
$ ssh-add -L | awk '{print $1" "$2}' | xclip
```

Or generate a new SSH key on the client and copy it to clipboard:

```console
$ ssh-keygen -f ~/.ssh/pcengines -C 'sysadm'
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in .ssh/pcengines.
Your public key has been saved in .ssh/pcengines.pub.

$ xclip ~/.ssh/pcengines.pub
```

On the APU, over the serial connection, as the primary user (e.g., `sysadm` - *not* `root`), configure SSH to accept that key by pasting it into `~/.ssh/authorized_keys`:

```console
$ mkdir ~/.ssh ; cat > ~/.ssh/authorized_keys
[Paste clipboard contents using the middle mouse button or Shift-Insert]
[Then press Control-D to save]
```

SSH from a client will now work:

```console
$ ssh sysadm@10.8.1.1 -i ~/.ssh/pcengines
Host key fingerprint is SHA256:AAAAA

Linux pcengines 4.9.0-8-amd64 #1 SMP Debian 4.9.130-2 (2018-10-27) x86_64
sysadm@pcengines~ %
```

Configure the connection on a client by editing `~/.ssh/config`:

```
Host pcengines
  HostName 10.8.1.1
  IdentityFile ~/.ssh/pcengines
  User sysadm
  Port 22
  ControlMaster auto
  ControlPath ~/.ssh/master-%r@%h:%p
  ControlPersist 1m
```

Connect using the new alias:

```console
$ ssh pcengines
Host key fingerprint is SHA256:AAAAA
Last login: Mon Jan 1 12:00:00 2018 from 10.8.1.2
sysadm@pcengines~ %
```

**Optional** Clone my configuration repository for the rest of the setup:

```console
$ git clone https://github.com/drduh/config
```

The serial connection can now be terminated. Be sure to log out with `Ctrl-D` or `exit` before disconnecting, otherwise anyone can plug in the serial cable to assume your session.

# DHCP and DNS

[Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) will provide [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) and handle DNS for the local network(s).

Use [drduh/config/dnsmasq.conf](https://github.com/drduh/config/blob/master/dnsmasq.conf) for a configuration example, including blocked domains:

```console
# cp config/dnsmasq.conf /etc/dnsmasq.conf

# cat config/domains/* | tee -a /etc/dnsmasq.conf

# vim /etc/dnsmasq.conf
```

Configure additional blocklist:

```console
$ git clone https://github.com/StevenBlack/hosts

$ sudo cp hosts/hosts /etc/dns-blocklist
```

## OpenBSD

To install dnsmasq as a service enabled on boot:

```console
$ doas pkg_add dnsmasq

$ doas rcctl start dnsmasq
dnsmasq(ok)

$ doas rcctl enable dnsmasq
```

# Wireless

## OpenBSD

**Note** Wireless performance is currently significantly worse on OpenBSD than Debian.

Edit `/etc/hostname.athn0` to include:

```shell
inet 192.168.1.1 255.255.255.0
media autoselect mode 11n mediaopt hostap chan 11
nwid NAME wpakey "PASSWORD"
```

Restart networking:

```console
$ doas sh /etc/netstart
```

## Debian

Install the default hostapd configuration:

```console
# zcat /usr/share/doc/hostapd/examples/hostapd.conf.gz | tee -a /etc/hostapd.conf
```

Or use [drduh/config/hostapd.conf](https://github.com/drduh/config/blob/master/hostapd.conf):

```console
# cp config/hostapd.conf /etc/hostapd.conf
```

Edit the configuration to set the network name and password.

*Tip* Avoid passwords with the characters `'` and `"`.

```console
# vim /etc/hostapd.conf
```

Ensure hostapd starts:

```console
# hostapd -dd /etc/hostapd.conf
```

**Note** If there are errors, download, compile and install a [newer version of hostapd](https://w1.fi/releases/?C=M;O=D):

```console
$ curl -O https://w1.fi/releases/hostapd-2.9.tar.gz

$ sha256sum hostapd-2.9.tar.gz
881d7d6a90b2428479288d64233151448f8990ab4958e0ecaca7eeb3c9db2bd7  hostapd-2.9.tar.gz

$ tar xf hostapd-2.9.tar.gz

$ cd hostapd-2.9/hostapd

$ sudo apt install pkg-config libnl-3-dev libssl-dev libnl-genl-3-dev
```

Edit the build configuration to enable features:

```console
$ cp defconfig .config

$ vim .config

# Uncomment these lines:
CONFIG_ACS=y
CONFIG_IEEE80211N=y
CONFIG_IEEE80211AC=y
```

Make, install and test the new version of hostapd:

```console
$ make -sj4

$ strip -s hostapd

$ sudo make install

$ hostapd -v
hostapd v2.9

$ sudo hostapd -d /etc/hostapd.conf
Configuration file: /etc/hostapd.conf
wlp5s0: interface state UNINITIALIZED->COUNTRY_UPDATE
ACS: Automatic channel selection started, this may take a bit
wlp5s0: interface state COUNTRY_UPDATE->ACS
wlp5s0: ACS-STARTED
wlp5s0: ACS-COMPLETED freq=5220 channel=44
Using interface wlp5s0 with hwaddr ca:9c:bc:8a:c7:5b and ssid "foo"
wlp5s0: interface state ACS->ENABLED
wlp5s0: AP-ENABLED
wlp5s0: STA c7:2d:df:b0:20:62 IEEE 802.11: authenticated
wlp5s0: STA c7:2d:df:b0:20:62 IEEE 802.11: associated (aid 1)
wlp5s0: AP-STA-CONNECTED c7:2d:df:b0:20:62
```

**Note** You may need to manually assign the interface an address:

```console
$ sudo ifconfig wlp5s0 192.168.1.1
```

# IP forwarding

In order to be a router, [IP forwarding](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) must be enabled.

## OpenBSD

Enable now and on boot:

```console
$ doas sysctl net.inet.ip.forwarding=1
net.inet.ip.forwarding: 0 -> 1

$ echo "net.inet.ip.forwarding=1" | doas tee -a /etc/sysctl.conf
```

## Debian

Enable now and on boot:

```console
# sysctl -w net.ipv4.ip_forward=1

# echo "net.ipv4.ip_forward=1" | tee --append /etc/sysctl.conf
```

# Configure firewall

## OpenBSD

See [PF - Building a Router](https://www.openbsd.org/faq/pf/example1.html), or use [drduh/config/pf](https://github.com/drduh/config/blob/master/pf/) files:

```console
$ doas mkdir /etc/pf

$ doas cp config/pf/pf.conf /etc/

$ doas cp config/pf/blocklist config/pf/martians config/pf/private /etc/pf/
```

Turn PF off and back on again:

```console
$ doas pfctl -d ; doas pfctl -e -f /etc/pf.conf
pf disabled
pf enabled 
```

**Optional** Use [drduh/config/scripts/pf-blocklist.sh](https://github.com/drduh/config/blob/master/scripts/pf-blocklist.sh) to find and block unwanted networks.

To inspect blocked traffic:

```console
$ doas tcpdump -ni pflog0
```

## Debian

Use [Iptables](https://en.wikipedia.org/wiki/Iptables) to manage a stateful firewall.

Use [drduh/config/scripts/iptables.sh](https://github.com/drduh/config/blob/master/scripts/iptables.sh) and edit it to your needs:

```console
# cp config/scripts/iptables.sh /etc

# vim /etc/iptables.sh

# chmod +x /etc/iptables.sh

# /etc/iptables.sh
```

Save the firewall rules to apply them on boot:

```console
# iptables-save | tee /etc/iptables/rules.v4
```

# Privoxy

[Privoxy](https://www.privoxy.org/) is a powerful Web proxy capable of filtering and rewriting URLs to block ads, upgrade HTTP connections, and more.

## Debian

Install Privoxy:

```console
# apt -y install privoxy
```

Use [drduh/config/privoxy/config](https://github.com/drduh/config/blob/master/privoxy/config) and [drduh/config/privoxy/user.action](https://github.com/drduh/config/blob/master/privoxy/user.action) - or edit the configuration yourself.

```console
# cp config/privoxy/config config/privoxy/user.action /etc/privoxy/
```

Restart the service and check the log:

```console
# service privoxy restart

# tail -f /var/log/privoxy/logfile
[...]
Info: Listening on port 8118 on IP address 127.0.0.1
Info: Listening on port 8118 on IP address 10.8.1.1
Info: Listening on port 8118 on IP address 172.16.1.1
Request: example.com/
Crunch: Redirected: http://bbc.com/
```

# Lighttpd

[Lighttpd](https://www.lighttpd.net/) with [mod_magnet](https://redmine.lighttpd.net/projects/1/wiki/Docs_ModMagnet) makes for a highly capable Web server which can be used to replace ad images with custom content, upload and share content on the local network, act as a captive portal, and more.

## Debian

Install Lighttpd with ModMagnet:

```console
# apt -y install lighttpd lighttpd-mod-magnet
```

Use [drduh/config/lighttpd/lighttpd.conf](https://github.com/drduh/config/blob/master/lighttpd/lighttpd.conf) and [drduh/config/lighttpd/magnet.luau](https://github.com/drduh/config/blob/master/lighttpd/magnet.luau) - or edit the configuration yourself.

```console
# cp config/lighttpd/lighttpd.conf config/lighttpd/magnet.luau /etc/lighttpd/
```

Restart the service and check the log:

```console
# service lighttpd restart

# cat /var/log/lighttpd/error.log
2019-01-01 12:00:00: (log.c.217) server started
```

# DNSCrypt

Download the Minisign [source code](https://github.com/jedisct1/minisign/releases/latest), build and install it:

```console
$ sudo apt install -y libsodium-dev pkg-config cmake

$ curl -o minisign-0.9.tar.gz -Lf https://github.com/jedisct1/minisign/archive/0.9.tar.gz

$ tar xf minisign-0.9.tar.gz

$ cd minisign-0.9

$ mkdir build

$ cd build

$ cmake ..

$ make

$ sudo make install
```

Download the latest Linux release - [`dnscrypt-proxy-linux_x86_64-*.tar.gz`](https://github.com/DNSCrypt/dnscrypt-proxy/releases/latest), verify it and edit the configuration:

```console
$ curl -LfO https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.1.0/dnscrypt-proxy-linux_x86_64-2.1.0.tar.gz

$ curl -LfO https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.1.0/dnscrypt-proxy-linux_x86_64-2.1.0.tar.gz.minisig

$ minisign -Vm dnscrypt-proxy-*.tar.gz -P RWTk1xXqcTODeYttYMCMLo0YJHaFEHn7a3akqHlb/7QvIQXHVPxKbjB5
Signature and comment signature verified
Trusted comment: timestamp:1628948801   file:dnscrypt-proxy-linux_x86_64-2.1.0.tar.gz

$ tar xf dnscrypt-proxy*.gz

$ cp config/dnscrypt-proxy.toml linux-x86_64/

$ cd linux-x86_64/

$ vim dnscrypt-proxy.toml
```

**Optional** Download and configure a hosts blacklist:

```console
$ git clone https://github.com/DNSCrypt/dnscrypt-proxy

$ cd dnscrypt-proxy/utils/generate-domains-blocklists

$ python3 generate-domains-blocklist.py > blocklist-$(date +%F).txt

$ cp blocklist-$(date +%F).txt ~/linux-x86_64/blocklist.txt
```

Start the program and check `dnscrypt.log` for success or errors:

```console
$ sudo ./dnscrypt-proxy
```

Once everything is working as expected, install and start dnscrypt-proxy as a service:

```console
$ sudo ./dnscrypt-proxy -service install

$ sudo ./dnscrypt-proxy -service start

$ tail -f dnscrypt.log
```

# Security and maintenance

To confirm the firewall is configured correctly, run port scans from an internal and external hosts, for example:

```console
$ nmap -v -A -T4 192.168.1.1 -Pn
```

To view blocked packets, tail the system message buffer on Linux:

```console
$ sudo dmesg -wH
[Jul 1 12:00] DROPIN>IN=enp1s0 OUT= MAC=00:00:00:00:00:00:00:00:00:00:00:00:00:00 SRC=192.168.1.10 DST=192.168.1.1 LEN=64 TOS=0x00 PREC=0x00 TTL=64 ID=29501 DF PROTO=TCP SPT=43228 DPT=554 WINDOW=16384 RES=0x00 SYN URGP=0
[...]
```

On OpenBSD, blocked packets will be sent to the PF log interface:

```console
$ doas tcpdump -ni pflog0
tcpdump: listening on pflog0, link-type PFLOG
12:00:00.000000 192.168.1.10.40770 > 192.168.1.1.1720: S 3331100898:3331180098(0) win 29200 <mss 1460,sackOK,timestamp 232580000 0,nop,wscale 7> (DF)
[...]
```

Install a USB camera and configure [Motion](https://motion-project.github.io/) to detect and monitor physical access.

(Linux only) Increase system entropy with a hardware device like [OneRNG](http://onerng.info/).

## OpenBSD

Check open network ports with `doas fstat | grep net` and `doas netstat -a -n -p udp -p tcp`.

Check running processes and sessions with `ps -A` and `last`.

Pay attention to [OpenBSD errata](https://www.openbsd.org/errata.html) and apply security fixes periodically with `doas syspatch`.

OpenBSD releases occur approximately every six months - [follow current snapshots](https://www.openbsd.org/faq/current.html) for faster updates by periodically running `doas sysupgrade -s` to reboot and install updates.

Check temperatures with `sysctl hw.sensors` or configure [sensorsd](https://man.openbsd.org/OpenBSD-current/man8/sensorsd.8).

## Debian

Check open ports and listening programs with `sudo lsof -Pni` and `sudo netstat -npl`.

Check running processes and logged-in users with `ps -eax` and `last -F`.

Pay attention to [Debian security advisories](https://lists.debian.org/debian-security-announce/recent) and run `sudo apt update && sudo apt upgrade` periodically or configure [unattended upgrades](https://wiki.debian.org/UnattendedUpgrades).

Install and enable [SELinux](https://wiki.debian.org/SELinux):

```console
# apt -y install selinux-basics selinux-policy-default

# selinux-activate

# reboot
```

Install and enable [AppArmor](https://wiki.debian.org/AppArmor), then reboot:

```console
# apt -y install apparmor apparmor-profiles apparmor-utils

# mkdir -p /etc/default/grub.d

# echo 'GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT apparmor=1 security=apparmor"' | tee /etc/default/grub.d/apparmor.cfg

# update-grub2 && reboot
```

Install and enable [Firejail](https://firejail.wordpress.com/):

```console
# apt -y install firejail firejail-profiles

# firecfg
```

See also [Debian SSD Optimizations](https://wiki.debian.org/SSDOptimization).

# Similar work

* [elad/openbsd-apu2](https://github.com/elad/openbsd-apu2)
* [martinbaillie/homebrew-openbsd-pcengines-router](https://github.com/martinbaillie/homebrew-openbsd-pcengines-router)
* [northox/openbsd-apu2](https://github.com/northox/openbsd-apu2)
* [vedetta-com/vedetta](https://github.com/vedetta-com/vedetta)

