This is a guide to building a router using the PC Engines [APU platform](http://www.pcengines.ch/apu.htm), the [Debian GNU/Linux](https://www.debian.org/releases/jessie/amd64/ch01s03.html.en) operating system for [network address translation](http://computer.howstuffworks.com/nat.htm), stateful firewalling, Web filtering, and much more.

I am **not** responsible for anything you do nor break by following any of these steps.

See also [drduh/Debian-Privacy-Server-Guide](https://github.com/drduh/Debian-Privacy-Server-Guide).

# Overview

The completed router configuration will enable:

* An egress Ethernet interface for Internet routing - can be connected to WAN or a cable modem
* A local Ethernet interface for low-trust devices - `10.8.1.0/24`
* A local Ethernet interface for trusted devices - `172.16.1.0/24`
* A local wireless interface for trusted devices - `192.168.1.0/24`

The following software will enable the router and enchance security and privacy:

* Dnsmasq to provide local DHCP and DNS with domain filtering
* DNSCrypt to encrypt outgoing DNS traffic to a trusted provider
* IPTables to enable NAT, enforce access control and block incoming traffic
* Privoxy to intercept and filter Web requests
* Lighttpd to serve static content locally

![network](https://user-images.githubusercontent.com/12475110/41503064-9e505032-717e-11e8-9b01-536abccd764d.png)

*Example network topology configured using this guide*

# Order hardware

Order hardware online from [PC Engines](http://www.pcengines.ch/order.htm) directly, or through a reseller.

Here is a suggested parts list (total cost - $185, including shipping):

* 1x apu2c4 APU.2C4 system board 4GB
* 1x case1d2bluu Enclosure 3 LAN, blue, USB
* 1x ac12vus2 AC adapter 12V US plug for IT equipment
* 1x msata16e SSD M-Sata 16GB MLC Phison
* 1x wle200nx Compex WLE200NX miniPCI express card
* 2x pigsma Cable I-PEX -> reverse SMA
* 2x antsmadb Antenna reverse SMA dual band

To connect over serial, you will also need something like [Sabrent USB 2.0 to Serial (9-Pin) Converter Cable](https://www.amazon.com/gp/product/B00IDSM6BW) and [Tripp Lite Null Modem Serial RS232 Cable](https://www.amazon.com/gp/product/B000067SCH).

# Assemble hardware

Clear an area to work. Unpack all the materials. Follow [apu cooling assembly instructions](http://www.pcengines.ch/apucool.htm) to install the heat conduction plate. Attach the mSATA disk, network card, antenna, and screw the enclosure shut.

Power on the board. To avoid arcing, plug in the DC jack first, then plug the adapter into mains.

By default, a memory test should run, and you should hear the board make a loud "beep" noise.

# Prepare Debian installer

Use another computer to prepare a netboot operating system installer.

Insert a USB drive. Run `dmesg` to identify the disk to use.

Erase and partition the disk using `cfdisk`, selecting `FAT16 (6)` as the partition type (must be under 4GB in size), and make it bootable:

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

**Optional** Install non-free firmware (only required by PC Engines APU1D (uses Realtek RTL8111E), not APU2C):

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

Unplug the drive and plug it into the APU.

# Connect over serial

On another computer, start [screen](https://www.gnu.org/software/screen/manual/screen.html):

    $ screen /dev/ttyUSB0 115200 8N1

Power up the APU board. Press `F10` and select USB boot.

Connect an Ethernet cable for Internet access.

Proceed through Debian installer, choosing the **internal hard drive** as the disk to partition. Also be sure to select it as the target for the boot loader installation.

Unplug the USB drive and reboot after installation completes.

# First boot

At the GRUB menu, you may get stuck at:

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

# First login

At the console prompt, log in, then escalate to root and install updates, and any necessary or optional software:

```
$ su
# apt-get update
# apt-get upgrade
# apt-get install -y \
    sudo ssh tmux lsof vim zsh tcpdump \
    dnsmasq privoxy hostapd \
    iptables iptables-persistent curl dnsutils ntp net-tools \
    make autoconf gcc gnupg ca-certificates apt-transport-https \
    man-db xclip screen minicom jmtpfs file feh scrot htop lshw
```

Update grub to use `nomodeset` from the previous step by editing `/etc/default/grub` and replacing `quiet` with `nomodeset console=ttyS0,115200n8`. Then run `sudo update-grub` to generate the GRUB configuration file.

If you'd like to enable "password-less" `sudo`, type `EDITOR=vi visudo` as the root user, and below the line:

    %sudo   ALL=(ALL:ALL) ALL

Add:

    sysadm     ALL=(ALL) NOPASSWD:ALL
    
Where `sysadm` is your primary user id.

Type `:x` to save and quit.

*(Optional)* Change the default login shell to `zsh`:

    # chsh -s /usr/bin/zsh sysadm
    
Press `Control-D` to exit `su` when finished.

# Configure network interfaces

At this point, either an Ethernet or wireless network interface (or both) can be set up to continue the rest of the guide over SSH instead of the serial terminal.

Connect an Ethernet cable between the middle port of the PC Engines board and another computer running Linux.

On both computers, determine the interface names available:

```
root@pcengines# lshw -C network | grep "logical name"
       logical name: enp1s0
       logical name: enp2s0
       logical name: enp3s0

user@localhost$ sudo lshw -C network | grep "logical name"
       logical name: eno1
       logical name: enp3s0
```

On the PC Engines host, edit `/etc/network/interfaces` to append:

```
allow-hotplug enp2s0
iface enp2s0 inet static
        address 10.8.1.1
        netmask 255.255.255.0
        gateway 10.8.1.1
```

Then restart networking and bring up the interface:

```
# service networking restart
# ifup enp2s0
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
auto wlan0
iface wlan0 inet static
address 192.168.1.1
netmask 255.255.255.0
hostapd /etc/hostapd.conf
```

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

# Configure DHCP and DNS

Use [Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) for [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) and DNS.

Edit `/etc/dnsmasq.conf` to include:

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

Restart the service:

    $ sudo service dnsmasq restart
    
See [drduh/config/dnsmasq.conf](https://github.com/drduh/config/blob/master/dnsmasq.conf) for additional options.

## Configure a Wireless Access Point

Install the default hostapd configuration:

    $ zcat /usr/share/doc/hostapd/examples/hostapd.conf.gz | sudo tee -a /etc/hostapd.conf

Edit `/etc/hostapd.conf` to include:

```
interface=wlan0
driver=nl80211
ssid="Hello, World!"
wpa=2
wpa_passphrase=super_secret_passphrase_999
```

Or download and use [drduh/config/hostapd.conf](https://github.com/drduh/config/blob/master/hostapd.conf):

    $ curl https://raw.githubusercontent.com/drduh/config/master/hostapd.conf | sudo tee /etc/hostapd.conf

Restart the service:

    $ sudo service hostapd restart

You may need to manually assign the interface an address:

    $ sudo ifconfig wlan0 192.168.1.1

# Enable IP forwarding

In order to be a router, [IP forwarding](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) must be enabled:

    $ sudo sysctl -w net.ipv4.ip_forward=1

To enable it permanently:

    $ echo "net.ipv4.ip_forward=1" | sudo tee --append /etc/sysctl.conf

# Configure the firewall

Use [IPTables](https://en.wikipedia.org/wiki/Iptables) to manage a stateful firewall. If you've never used IPTables before, start with the following shell script and edit it to suit your needs.

Download and edit [drduh/config/firewall.sh](https://github.com/drduh/config/blob/master/firewall.sh):

    $ curl -LfvO https://raw.githubusercontent.com/drduh/config/master/firewall.sh

Use `chmod +x iptables.sh` to make the script executable and apply it with `sudo ./iptables.sh`.

The PC Engines router should now be able to route packets. Confirm by sending an outbound ping from another computer connected to the router.

To make the firewall rules permanent:

    $ sudo iptables-save | sudo tee /etc/iptables/rules.v4

Either install or `dpkg-reconfigure` the `iptables-persistent` package, or manually restore the rules on boot, for example by editing `/etc/rc.local`:

```
#!/bin/sh -e

# restore firewall rules
sudo /sbin/iptables-restore < /etc/iptables/rules.v4

exit 0
```

# Web filtering

Install [Privoxy](https://www.privoxy.org/) and [Lighttpd](https://www.lighttpd.net/) with [mod_magnet](https://redmine.lighttpd.net/projects/1/wiki/Docs_ModMagnet)

    $ sudo apt-get install -y privoxy lighttpd lighttpd-mod-magnet

Download and edit [drduh/config/lighttpd.conf](https://github.com/drduh/config/blob/master/lighttpd.conf), [drduh/config/magnet.luau](https://github.com/drduh/config/blob/master/magnet.luau) and [drduh/config/privoxy](https://github.com/drduh/config/blob/master/privoxy):

```
$ curl https://raw.githubusercontent.com/drduh/config/master/lighttpd.conf | sudo tee /etc/lighttpd/lighttpd.conf

$ sudo vim /etc/lighttpd/lighttpd.conf

$ curl https://raw.githubusercontent.com/drduh/config/master/magnet.luau | sudo tee /etc/lighttpd/magnet.luau

$ sudo vim /etc/lighttpd/magnet.luau

$ curl https://raw.githubusercontent.com/drduh/config/master/privoxy | sudo tee /etc/privoxy/config

$ sudo vim /etc/privoxy/config
```

Restart both services and check to make sure they work.

```
$ sudo service lighttpd restart
$ sudo service privoxy restart
```

To set an image blocker, edit `/etc/privoxy/user.action` to add the following to the bottom:

```
{+set-image-blocker{http://10.8.1.1/}}
/.*.[jpg|jpeg|gif|png|tif|tiff]$
```

# Security and maintenance

To confirm the firewall is configured correctly, run a port scan from an external host or over the Tor network using [haad/proxychains](https://github.com/haad/proxychains):

    $ nmap -v -A -T4 xxx.xxx.xxx.xxx -Pn

Pay attention to [Debian security advisories](https://lists.debian.org/debian-security-announce/recent) and log in to `apt-get update && apt-get upgrade` periodically - or configure automatic updates.

So long as no ports/services are exposed to the Internet interface, the risk of compromise is minimal. Nevertheless, it's good practice to occassionally check running processes (`ps -ef`), open network connections (`sudo lsof -Pni`) and remote access (`w` and `last`), as well as any suspicious files in `/tmp` and elsewhere.

# Similar work

* [elad/openbsd-apu2](https://github.com/elad/openbsd-apu2)
* [martinbaillie/homebrew-openbsd-pcengines-router](https://github.com/martinbaillie/homebrew-openbsd-pcengines-router)
