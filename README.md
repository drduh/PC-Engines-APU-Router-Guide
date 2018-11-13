This is a guide to building a router using the PC Engines [APU platform](http://www.pcengines.ch/apu.htm) and an operating system like [Debian](https://www.debian.org/releases/jessie/amd64/ch01s03.html.en) or [OpenBSD](https://www.openbsd.org/) for [network address translation](http://computer.howstuffworks.com/nat.htm), stateful firewalling, Web traffic filtering, and much more.

I am **not** responsible for anything you do nor break by following any of these steps.

See also [drduh/Debian-Privacy-Server-Guide](https://github.com/drduh/Debian-Privacy-Server-Guide).

# Overview

The completed router configuration will enable:

* An egress Ethernet interface for Internet routing - can be connected to WAN or a cable modem
* A local wireless interface for trusted devices - `192.168.1.0/24`
* A local Ethernet interface for low-trust devices - `10.8.1.0/24`
* A local Ethernet interface for trusted devices - `172.16.1.0/24`
* An additonal (4th) Ethernet interface is available on APU4+

The following software will enable the router and enchance security and privacy:

* Dnsmasq to provide local DHCP and DNS with domain filtering
* DNSCrypt to encrypt outgoing DNS traffic to a trusted provider
* IPTables to enable NAT, enforce access control and block incoming traffic
* Privoxy to intercept and filter Web requests
* Lighttpd to serve static content locally

![network](https://user-images.githubusercontent.com/12475110/41503064-9e505032-717e-11e8-9b01-536abccd764d.png)

*Example network topology configured using this guide*

# Hardware

Order hardware online from [PC Engines](https://www.pcengines.ch/order.htm) directly, or through a reseller.

Here is a suggested parts list:

| Quantity | Part | Description | Cost
|----------|------|-------------|------
| 1 | apu2c4 | APU.2C4 system board 4GB | $132.00
| 1 | case1d2bluu | Enclosure 3 LAN, blue | $10.00
| 1 | ac12vus2 | AC adapter 12V 2A US plug | $4.40
| 1 | msata16g | SSD M-Sata 16GB MLC, Phison S11 | $17.80
| 1 | wle200nx | Compex WLE200NX miniPCI express card | $19.00
| 2 | pigsma | Cable I-PEX -> reverse SMA | $3.00
| 2 | antsmadb | Antenna reverse SMA dual band | $4.10

To connect over serial, you will need a [USB to Serial (9-Pin) Converter Cable](https://www.amazon.com/gp/product/B00IDSM6BW) and [Modem Serial RS232 Cable](https://www.amazon.com/gp/product/B000067SCH).

# Assemble hardware

Read over the APU2 series [board reference](http://www.pcengines.ch/pdf/apu2.pdf) document before starting.

Clear an area to work. Unpack all the materials. Follow [apu cooling assembly instructions](https://www.pcengines.ch/apucool.htm) to install the heat conduction plate.

Attach the mSATA disk and miniPCI wireless adapter.

**Note** Wireless radio cards are ESD sensitive, especially the RF switch and the power amplifier. To avoid damage by electrostatic discharge, the following installation procedure is [recommended](https://www.pcengines.ch/wle200nx.htm):

> 1. Touch your hands and the bag containing the radio card to a ground point on the router board (for example one of the mounting holes). This will equalize the potential of radio card and router board.
> 2. Install the radio card in the miniPCI express socket.
> 3. Install the pigtail cable in the cut-out of the enclosure. This will ground the pigtail to the enclosure.
> 4. Touch the I-PEX connector of the pigtail to the mounting hole (discharge), then plug onto the radio card (this is where the pre-requisite patience comes in.

Power on the board. To avoid arcing, plug in the DC jack first, then plug the adapter into mains.

By default, a memory test should run, and you should hear the board make a loud "beep" noise. If not, reboot and press `F10` to select `Payload [memtest]` and complete at least one pass.

# Prepare OS installation

Use another computer to prepare a operating system installer.

Insert a USB disk. Run `dmesg` to identify it.

## Debian

### Easy way

Download the latest [`netinst.iso`](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/) image and copy it to the USB disk:

    $ sudo dd if=debian-9.6.0-amd64-netinst.iso of=/dev/sdd bs=1M

### Manual way

Erase and partition the USB disk using `cfdisk`, selecting `FAT16 (6)` as the partition type (must be under 4GB in size), and make it bootable:

    $ sudo cfdisk /dev/sdd
    
You can also use a GUI-based software like `gparted` for the same.

Install the master boot record:

    $ sudo install-mbr /dev/sdd
    
Install the package `dosfstools` to create the proper filesystem:

    $ sudo mkfs.fat -F 16 -I /dev/sdd1

Install the [SYSLINUX bootloader](http://www.syslinux.org/wiki/index.php?title=Install):

    $ sudo syslinux /dev/sdd1

Mount the new filesystem:

    $ sudo mkdir /mnt/usb

    $ sudo mount /dev/sdd1 /mnt/usb

    $ cd /mnt/usb

Download netboot installer files:

    $ sudo curl -LfvO http://ftp.debian.org/debian/dists/stable/main/installer-amd64/current/images/netboot/debian-installer/amd64/initrd.gz

    $ sudo curl -LfvO http://ftp.debian.org/debian/dists/stable/main/installer-amd64/current/images/netboot/debian-installer/amd64/linux

**Optional** Install non-free firmware (only required by APU1D (uses Realtek RTL8111E), not APU2C):

    $ sudo curl -LfvO https://cdimage.debian.org/cdimage/unofficial/non-free/firmware/stable/current/firmware.zip

Verify file integrity:

```
$ shasum -a 256 linux initrd.gz firmware.zip
bea409469a665a930954c7ee1bbaa77964357988163d83a50ff741d1bbba0811  linux
362be3860fc427b3fe81071741d56863b37a8fc05e3126142559782a9b9d0749  initrd.gz
0aaee0d6388fc626317b0f248b9bd1d0954f847707a3d0e377e82761fd45aa41  firmware.zip
```

In the same `/mnt/usb` directory, create a file called `syslinux.cfg` with the following contents:

```
default linux
timeout 100
prompt 1
label linux
kernel linux
append vga=off initrd=initrd.gz console=ttyS0,115200n8 fb=false
```

The directory contents should contain:

    $ ls /mnt/usb
    firmware.zip  initrd.gz  ldlinux.c32  ldlinux.sys  linux  syslinux.cfg

Unmount the drive:

    $ cd

    $ sudo umount /mnt/usb

Unplug the USB disk and plug it into the APU.

## OpenBSD

Download the latest OpenBSD installer, [`amd64/install64.fs`](https://cloudflare.cdn.openbsd.org/pub/OpenBSD/6.4/amd64/install64.fs) and copy the file to the USB device:

    $ sudo dd if=install64.fs of=/dev/sdd bs=1M

# Connect over serial

To connect from another computer running Linux, start [screen](https://www.gnu.org/software/screen/manual/screen.html). The APU uses 115200 baud rate, 8N1 (8 data bits, no parity, 1 stop bit):

    $ screen /dev/ttyUSB0 115200 8N1

Or from OpenBSD, use [cu](https://man.openbsd.org/cu):

    $ doas cu -r -s 115200 -l cuaU0

Power up the APU board, DC jack first. Make note of the BIOS version displayed briefly before POST.

# Updating BIOS

If the version is [out of date](https://pcengines.github.io/), download and extract [TinyCore Linux](http://pcengines.ch/file/apu2-tinycore6.4.img.gz) and latest BIOS release. Mount a USB disk and write the TinyCore image, then copy the latest firmware:

```shell
$ sudo dd if=apu2-tinycore6.4.img of=/dev/sdd bs=1M
511+0 records in
511+0 records out
535822336 bytes (536 MB, 511 MiB) copied, 22.9026 s, 23.4 MB/s

$ sudo mount /dev/sdd1 /mnt/usb

$ sudo cp apu4_v4.8.0.2.rom /mnt/usb

$ sudo umount /mnt/usb
```

Unplug the USB disk and connect it to the APU, then boot to it using `F10`.

Flash the BIOS:

```shell
root@box:~# flashrom -p internal -w /media/SYSLINUX/apu4_v4.8.0.2.rom
```

![Using flashrom](https://user-images.githubusercontent.com/12475110/43032327-eba5b3ea-8c68-11e8-8ded-8d1d0eb32720.png)
*Specifying "boardmismatch" was necessary with a newer APU4 model*

Verify the correct BIOS version right after rebooting.

# Installing the OS

Press `F10` and select USB boot.

## Debian

Proceed through Debian installer, choosing the **internal hard drive** as the disk to partition. Also be sure to select it as the target for the boot loader installation.

## OpenBSD

Set serial console parameters before starting installation:

```shell
>> OpenBSD/amd64 BOOT 3.34
boot> stty com0 115200
boot> set tty com0
switching console to com>> OpenBSD/amd64 BOOT 3.34
boot> [Press Enter]
```

When presented with a list of network interfaces, `em0` is the one closest to the serial port:

    Available network interfaces are: em0 em1 em2 em3 vlan0.

To mount `/tmp` as `tmpfs` (in memory only), drop to a shell with `S` once installation completes and append the following to `/etc/fstab`:

    swap /tmp mfs rw,noexec,nosuid,nodev,-s=256M 0 0

# First boot

## Debian

After installation, at the GRUB menu, you may get stuck at:

```
Loading Linux 4.9.0-6-amd64 ...
Loading initial ramdisk ...
```

If this is the case, press `e` at the kernel selection screen in GRUB, scroll down and replace the word `quiet` with:

    nomodeset console=ttyS0,115200n8
    
**Note** If arrow keys do not work in GRUB, try using emacs key bindings to navigate the text field:

* `Control-B` to move left
* `Control-F` to move right
* `Control-P` to move up
* `Control-N` to move down

Press `Control-X` to continue booting. You should see verbose boot messages appear on the screen.

**Note** If you get an error like, `Alert! /dev/sdX1 does not exist dropping to shell` and are dropped to an initramfs prompt, reboot and edit the `quiet` line to point to `/dev/sda1` or correct partition.

## OpenBSD

The following parameters should have been appended to `/etc/boot.conf` by the installer:

    stty com0 115200
    set tty com0

# First login

## Debian

Login as `root`, install and update any software that may be needed:

```
# apt-get update
# apt-get -y upgrade
# apt-get -y install \
    sudo ssh tmux lsof vim zsh tcpdump \
    dnsmasq privoxy hostapd nmap \
    iptables iptables-persistent curl dnsutils ntp net-tools \
    make autoconf gcc gnupg ca-certificates apt-transport-https \
    man-db xclip screen minicom jmtpfs file feh scrot htop lshw less
```

Update GRUB to use `nomodeset` from the previous step by editing `/etc/default/grub` and replacing `quiet` with `nomodeset console=ttyS0,115200n8`. Then run `sudo update-grub` to generate the GRUB configuration file.

To enable password-less `sudo`, type `EDITOR=vi visudo` as the root user, and below the line:

    %sudo   ALL=(ALL:ALL) ALL

Add:

    sysadm     ALL=(ALL) NOPASSWD:ALL
    
Where `sysadm` is your primary user id.

Type `:x` to save and quit.

*(Optional)* Change the default login shell to `zsh`:

    # chsh -s /usr/bin/zsh sysadm

## OpenBSD

Login as `root`, install any pending updates with [`syspatch`](https://man.openbsd.org/syspatch).

    # syspatch

Install any needed software:

    # pkg_add -Vv vim zsh nmap curl pftop vnstat

Edit `/etc/doas.conf` to allow the regular user to run [privileged commands](https://man.openbsd.org/doas.conf). For a similar password-less setup to the section above, use:

```
permit nopass keepenv :wheel
permit nopass keepenv root
```

# Configure network interfaces

At this point, either an Ethernet or wireless network interface (or both) can be set up to continue the rest of the guide over SSH instead of the serial terminal.

## Debian

From the APU and another Linux computer, determine the interface names available:

```shell
root@pcengines# lshw -C network | grep "logical name"
       logical name: enp1s0
       logical name: enp2s0
       logical name: enp3s0
       logical name: wlp4s0

user@localhost$ sudo lshw -C network | grep "logical name"
       logical name: eno1
       logical name: enp3s0
```

On the APU, edit `/etc/network/interfaces` to append:

```
allow-hotplug enp2s0
iface enp2s0 inet static
address 10.8.1.1
netmask 255.255.255.0
gateway 10.8.1.1
```

Then restart networking and bring up the interface:

```
$ sudo service networking restart
$ sudo ifup enp2s0
```

On the other Linux computer, append:

```
allow-hotplug eno1
iface eno1 inet static
address 10.8.1.2
netmask 255.255.255.0
gateway 10.8.1.1
```

Then also restart networking and bring up the interface:

```
$ sudo service networking restart
$ sudo ifup eno1
```

It should now be possible to ping the router:

```
$ ping -c 1 10.8.1.1
PING 10.8.1.1 (10.8.1.1) 56(84) bytes of data.
64 bytes from 10.8.1.1: icmp_seq=1 ttl=64 time=0.693 ms
```

To configure the wireless interface, edit `/etc/network/interfaces` to include:

```
auto wlp4s0
iface wlp4s0 inet static
address 192.168.1.1
netmask 255.255.255.0
hostapd /etc/hostapd.conf
```

## OpenBSD

On the APU, set a local network interface address:

    $ doas ifconfig em1 10.8.1.1 255.255.255.0

Make it permanent:

    $ echo "inet 10.8.1.1 255.255.255.0" | doas tee /etc/hostname.em1

Configure a client to connect using the section above or set up DHCP by following the [Networking FAQ](https://www.openbsd.org/faq/faq6.html).

# Configure SSH

An SSH connection to the router should now be able to be established, but not yet authorized:

```
$ ssh sysadm@10.8.1.1
The authenticity of host '10.8.1.1 (10.8.1.1)' can't be established.
ECDSA key fingerprint is SHA256:AAAAA.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.8.1.1' (ECDSA) to the list of known hosts.
Permission denied (publickey,password).
```

Generate an SSH keypair on a computer that will be used to login to the router:

```
$ ssh-keygen -f ~/.ssh/pcengines -C 'sysadm'
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in .ssh/pcengines.
Your public key has been saved in .ssh/pcengines.pub.
```

Copy the public key to clipboard:

```
$ cat ~/.ssh/pcengines.pub | xclip
```

Over the serial connection, as the configured user (e.g. `sysadm` - *not* `root`) on the router, configure SSH to accept that key:

```
$ mkdir ~/.ssh
$ cat > ~/.ssh/authorized_keys
[Paste the copied public key using the middle mouse button]
[Press Control-D to save]
```

The SSH connection should now be accepted by specifying the private key:

```
$ ssh sysadm@10.8.1.1 -i ~/.ssh/pcengines
Host key fingerprint is SHA256:AAAAA

Linux pcengines1 4.9.0-6-amd64 #1 SMP Debian 4.9.88-1+deb9u1 (2018-05-07) x86_64
pcengines%
```

To make connecting and copying files over easier, append the following to `~/.ssh/config` on a client:

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

To connect:

```
$ ssh pcengines
Host key fingerprint is SHA256:AAAAA
Linux pcengines1 4.9.0-6-amd64 #1 SMP Debian 4.9.88-1+deb9u1 (2018-05-07) x86_64
Last login: Mon Jan 1 12:00:00 2018 from 10.8.1.2
pcengines%
```

To copy files:

```
$ scp .tmux.conf .vimrc .zshrc pcengines:~
```

See [drduh/YubiKey-Guide](https://github.com/drduh/YubiKey-Guide) to better protect the SSH key and make it portable.

# DHCP and DNS

[Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) can be used to provide [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) and DNS. See [drduh/config/dnsmasq.conf](https://github.com/drduh/config/blob/master/dnsmasq.conf) for recommended options. Edit `/etc/dnsmasq.conf` to include at least the following:

```
dhcp-lease-max=15
dhcp-option=option:router,192.168.1.1
dhcp-range=192.168.1.2,192.168.1.15,24h
domain-needed
domain=whatever
interface=wlan0
listen-address=127.0.0.1,192.168.1.1
local=/whatever/
log-async=5
log-dhcp
log-facility=/var/log/dnsmasq
server=8.8.8.8
stop-dns-rebind
bogus-priv
```

## Debian

Restart the service:

    $ sudo service dnsmasq restart

Enable on boot:

    $ sudo update-rc.d dnsmasq enable

## OpenBSD

Start the service:

    $ doas rcctl start dnsmasq
    dnsmasq(ok)

Enable on boot:

    $ doas rcctl enable dnsmasq
    

# Wireless Access Point

## Debian

Install the default hostapd configuration:

    $ zcat /usr/share/doc/hostapd/examples/hostapd.conf.gz | sudo tee -a /etc/hostapd.conf

Or download and use [drduh/config/hostapd.conf](https://github.com/drduh/config/blob/master/hostapd.conf):

    $ curl https://raw.githubusercontent.com/drduh/config/master/hostapd.conf | sudo tee /etc/hostapd.conf

At a minimum, it should include:

```
interface=wlan0
driver=nl80211
ssid="Hello, World!"
wpa=2
wpa_passphrase=super_secret_passphrase_999
```

Restart the service:

    $ sudo service hostapd restart

You may need to manually assign the interface an address manually:

    $ sudo ifconfig wlan0 192.168.1.1

# IP forwarding

In order to be a router, [IP forwarding](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) must be enabled.

## Debian

Enable now:

    $ sudo sysctl -w net.ipv4.ip_forward=1

Enable permanently:

    $ echo "net.ipv4.ip_forward=1" | sudo tee --append /etc/sysctl.conf

## OpenBSD

    $ echo 'net.inet.ip.forwarding=1' | doas tee -a /etc/sysctl.conf

# Configure firewall

## Debian

Use [IPTables](https://en.wikipedia.org/wiki/Iptables) to manage a stateful firewall. If you've never used IPTables before, start with the following shell script and edit it to suit your needs.

Download and edit [drduh/config/iptables.sh](https://github.com/drduh/config/blob/master/iptables.sh):

    $ curl -LfvO https://raw.githubusercontent.com/drduh/config/master/iptables.sh

Use `chmod +x iptables.sh` to make the script executable and apply with `sudo ./iptables.sh`.

The APU router should now be able to route packets. Confirm by sending an outbound ping from another computer connected to the router.

To make the firewall rules permanent:

    $ sudo iptables-save | sudo tee /etc/iptables/rules.v4

Either install or `dpkg-reconfigure` the `iptables-persistent` package, or manually restore the rules on boot, for example by editing `/etc/rc.local`:

```
#!/bin/sh -e

# restore firewall rules
sudo /sbin/iptables-restore < /etc/iptables/rules.v4

exit 0
```

## OpenBSD

Download and edit [drduh/config/pf/](https://github.com/drduh/config/blob/master/pf/):

    $ doas curl -Lfvo /etc/pf.conf https://raw.githubusercontent.com/drduh/config/master/pf/pf.conf
    $ doas mkdir /etc/pf
    $ doas curl -Lfvo /etc/pf/blacklist https://raw.githubusercontent.com/drduh/config/master/pf/blacklist
    $ doas curl -Lfvo /etc/pf/martians https://raw.githubusercontent.com/drduh/config/master/pf/martians
    $ doas curl -Lfvo /etc/pf/private https://raw.githubusercontent.com/drduh/config/master/pf/private

Turn pf off and back on again:

    $ doas pfctl -d ; doas pfctl -e -f /etc/pf.conf
    pf disabled
    pf enabled 

See also [PF - Building a Router](https://www.openbsd.org/faq/pf/example1.html).

# Web filtering

[Privoxy](https://www.privoxy.org/) is a powerful Web proxy capable of filtering and rewriting URLs. It may be used in combination with [Lighttpd](https://www.lighttpd.net/) with [mod_magnet](https://redmine.lighttpd.net/projects/1/wiki/Docs_ModMagnet) to block or replace ads/content with custom images, for example.

## Debian

Install required software:

    $ sudo apt-get -y install privoxy lighttpd lighttpd-mod-magnet

Edit the default configurations, or download and edit:

* [drduh/config/lighttpd.conf](https://github.com/drduh/config/blob/master/lighttpd.conf)
* [drduh/config/magnet.luau](https://github.com/drduh/config/blob/master/magnet.luau)
* [drduh/config/privoxy](https://github.com/drduh/config/blob/master/privoxy)
* [drduh/config/user.action](https://github.com/drduh/config/blob/master/user.action)

```
$ curl https://raw.githubusercontent.com/drduh/config/master/lighttpd.conf | sudo tee /etc/lighttpd/lighttpd.conf
$ curl https://raw.githubusercontent.com/drduh/config/master/magnet.luau | sudo tee /etc/lighttpd/magnet.luau
$ curl https://raw.githubusercontent.com/drduh/config/master/privoxy | sudo tee /etc/privoxy/config
$ curl https://raw.githubusercontent.com/drduh/config/master/user.action | sudo tee /etc/privoxy/user.action
```

Restart both services and check to make sure they work:

```
$ sudo service lighttpd restart

$ sudo service privoxy restart
```

# Security and maintenance

To confirm the firewall is configured correctly, run a port scan from an external host or over the Tor network using [haad/proxychains](https://github.com/haad/proxychains):

    $ nmap -v -A -T4 1.2.3.4 -Pn

So long as no services are exposed to the Internet interface, the risk of *remote* compromise is minimal (physical access and access is not addressed in this guide).

Install a USB camera and configure [Motion](https://motion-project.github.io/) to monitor and detect physical access.

## Debian

Pay attention to [Debian security advisories](https://lists.debian.org/debian-security-announce/recent).

Run `apt-get update && apt-get upgrade` periodically, or configure [unattended upgrades](https://wiki.debian.org/UnattendedUpgrades).

See also [Debian SSD Optimizations](https://wiki.debian.org/SSDOptimization)

## OpenBSD

Pay attention to [OpenBSD errata](https://www.openbsd.org/errata.html).

OpenBSD releases occur approximately every six months - [follow current snapshots](https://www.openbsd.org/faq/current.html) for faster updates.

# Similar work

* [elad/openbsd-apu2](https://github.com/elad/openbsd-apu2)
* [martinbaillie/homebrew-openbsd-pcengines-router](https://github.com/martinbaillie/homebrew-openbsd-pcengines-router)
* [vedetta-com/vedetta](https://github.com/vedetta-com/vedetta)
* [northox/openbsd-apu2](https://github.com/northox/openbsd-apu2)

