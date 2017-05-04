**This guide is in-progress and not yet complete. Use at your own risk!**

This is a hands-on guide to building your own router using a PC Engines [APU platform](http://www.pcengines.ch/apu.htm), installing a [Debian GNU/Linux](https://www.debian.org/releases/jessie/amd64/ch01s03.html.en) operating system and using it for [network address translation](http://computer.howstuffworks.com/nat.htm), stateful firewalling, Web filtering, and much more.

It is **not** meant to be a canonical guide on best practices; I just like setting up my router and firewall in these ways.

I am **not** responsible for anything you do nor break by following any of these steps.

See also [drduh/Debian-Privacy-Server-Guide](https://github.com/drduh/Debian-Privacy-Server-Guide) which is similar to this guide, but is written for a remote server, and can be used in combination with this guide.

# Overview

The completed router configuration will enable: 

* 1x egress Ethernet interface for Internet routing - connected to WAN or a cable modem
* 1x "trusted" Ethernet interface for local area networking - `10.8.1.0/24`
* 1x "less-trusted" ([DMZ](https://en.wikipedia.org/wiki/DMZ_(computing))) Ethernet interface for local area networking - `172.16.1.0/24`
* 1x Wireless interface for local area networking - `192.168.1.0/24`

As well as the following services (some optional, of course):

* Dnsmasq to provide local DHCP and DNS
* DNSCrypt to encrypt outgoing DNS traffic
* IPTables to enable NAT, enforce access control and filter traffic
* Privoxy to intercept and filter Web requests
* Lighttpd to serve static content, for example to replace unwanted ad images

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

Use another computer to prepare an operating system installer. I am using Debian 8 "Jessie" to prepare a Debian installer.

Insert a USB drive. Run `dmesg` to identify the disk to use.

Erase and partition the disk, selecting `FAT16` as the partition type, and make it bootable:

    $ sudo cfdisk /dev/sdd

Install the [SYSLINUX bootloader](http://www.syslinux.org/wiki/index.php?title=Install):

    $ sudo syslinux /dev/sdd1

Mount the new filesystem (assuming USB disk is `/dev/sdd`):

    $ sudo mkdir /mnt/usb

    $ sudo mount /dev/sdd1 /mnt/usb

    $ cd /mnt/usb

Download netboot installer files:

    $ sudo curl -LfvO http://ftp.debian.org/debian/dists/jessie/main/installer-amd64/current/images/netboot/debian-installer/amd64/initrd.gz

    $ sudo curl -LfvO http://ftp.debian.org/debian/dists/jessie/main/installer-amd64/current/images/netboot/debian-installer/amd64/linux

**(Optional)** Install non-free firmware (*apparently* needed by APU1D, but not APU2C):

    $ curl -LfvO http://cdimage.debian.org/cdimage/unofficial/non-free/firmware/jessie/current/firmware.zip

Verify file integrity:

```
$ shasum -a 256 linux initrd.gz
599cb2358e8bfbbeea939978bab7054a7681304fdfb1469ad34a815356c8bf6a  linux
ee69db088188d995f72f29c359ca2f09b7d011a0ec11592c35e18ca0c317c164  initrd.gz
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

Unmount the drive:

    $ cd

    $ sudo umount /mnt/usb

Unplug the drive and plug it into the APU.

# Connect over serial

On a client machine, start [screen](https://www.gnu.org/software/screen/manual/screen.html):

    $ screen /dev/ttyUSB0 115200 8N1

Power up the APU board. Press `F10` and select USB boot.

Connect an Ethernet cable for Internet access.

Proceed through Debian installer, choosing the **internal mSATA hard drive** and GRUB installer selection appropriately (be careful *not* to select the USB drive!).

Reboot after installation completes.

# First boot

At the GRUB menu, you may get stuck at:

```
Loading Linux 3.16.0-4-amd64 ...
Loading initial ramdisk ...
```

If this is the case, press `e` at the kernel selection screen in GRUB, scroll down and replace the word `quiet` with:

    nomodeset console=ttyS0,115200n8

Press `Control-X` to continue booting. You should see verbose boot messages appear on the screen.

**Note** To make this fix permanent later, edit `/etc/default/grub` and replace `quiet` with `nomodeset console=ttyS0,115200n8`, then run `sudo update-grub`

# First login

At the console prompt, log in, then escalate to root and install updates, and any necessary software:

```
$ su
# apt-get update
# apt-get upgrade
# apt-get install tmux lsof vim zsh tcpdump iptables iptables-persistent sudo ssh curl dnsutils ntp make autoconf gcc gnupg gnupg-curl
```

If you'd like to enable "password-less" `sudo`, type `EDITOR=vi visudo` as the root user, and below the line:

    %sudo   ALL=(ALL:ALL) ALL

Add:

    sysadm     ALL=(ALL) NOPASSWD:ALL

Type `:x` to save and quit.

Press `Control-D` to exit `su`.

**(Optional)** Type `chsh` to change the login shell to Z shell (zsh).

Depending on your current network configuration, you may be able to connect over ssh to finish setting up Debian, or continue using serial console.

With ssh, you can quickly copy over configuration and build files:

    $ scp .tmux.conf .zshrc .vimrc sysadm@10.8.8.10:~

# Configure network interfaces

Edit `/etc/network/interfaces`:

```
auto wlan0
iface wlan0 inet static
address 192.168.1.1
netmask 255.255.255.0
# uncomment the following line to enable the Access Point daemon for Wi-Fi
#hostapd /etc/hostapd.conf
```

# Configure DHCP and DNS

Use [Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) for [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) and DNS.

Edit `/etc/dnsmasq.conf` to include something like:

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

## Configure a Wireless Access Point

**(Optional)** Install the default configuration:

    $ zcat /usr/share/doc/hostapd/examples/hostapd.conf.gz | sudo tee -a /etc/hostapd.conf

Edit `/etc/hostapd.conf` to include something like:

```
interface=wlan0
driver=nl80211
ssid="Hello, World!"
wpa=2
wpa_passphrase=super_secret_passphrase_999
```

Restart the service:

    $ sudo service hostapd restart

You may need to manually assign the interface an address:

    $ sudo ifconfig wlan0 192.168.1.1

# Enable IP forwarding

In order to be a router, IP forwarding must be enabled:

    $ sudo sysctl -w net.ipv4.ip_forward=1

To enable it permanently:

    $ echo "net.ipv4.ip_forward = 1" | sudo tee --append /etc/sysctl.conf

# Configure the firewall

Use [IPTables](https://en.wikipedia.org/wiki/Iptables) to manage a stateful firewall.

If you've never used IPTables before, start with the following shell script and edit it to suit your needs.

Create a file called `firewall.sh` with the following contents:

```
#!/bin/bash

PATH='/sbin'

EXT=eth0
INT=eth1
DMZ=eth2
WIFI=wlan0

INT_NET=172.16.1.0/24
DMZ_NET=10.8.4.0/24
WIFI_NET=192.168.1.0/24

echo "Flush rules"
iptables -F
iptables -t nat -F
iptables -X
iptables -Z

iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

echo "Allow loopback"
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

echo "Drop invalid states"
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
iptables -A OUTPUT -m conntrack --ctstate INVALID -j DROP
iptables -A FORWARD -m conntrack --ctstate INVALID -j DROP

echo "Allow established, related and ICMP echo packets"
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p icmp -m icmp --icmp-type 8 -m conntrack --ctstate NEW -j ACCEPT

echo "Allow DHCP"
iptables -I INPUT -i $DMZ -p udp -m udp --dport 67 -m conntrack --ctstate NEW -j ACCEPT
iptables -I INPUT -i $INT -p udp -m udp --dport 67 -m conntrack --ctstate NEW -j ACCEPT
iptables -I INPUT -i $WIFI -p udp -m udp --dport 67 -m conntrack --ctstate NEW -j ACCEPT

echo "Allow ssh from trusted Ethernet"
iptables -A INPUT -i $INT -s $INT_NET -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT

echo "Allow DNS (TCP and UDP)"
iptables -A INPUT -i $INT -s $INT_NET -p udp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -i $INT -s $INT_NET -p tcp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -i $DMZ -s $DMZ_NET -p udp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -i $DMZ -s $DMZ_NET -p tcp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -i $WIFI -s $WIFI_NET -p udp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -i $WIFI -s $WIFI_NET -p tcp --dport 53 -m conntrack --ctstate NEW -j ACCEPT

echo "Add transparent Privoxy forwarding"
iptables -A INPUT -i $INT -s $INT_NET -p tcp --dport 8118 -m conntrack --ctstate NEW -j ACCEPT
iptables -t nat -A PREROUTING -i $INT -p tcp --dport 80 -j DNAT --to-destination 10.8.1.1:8118
iptables -A INPUT -i $DMZ -s $DMZ_NET -p tcp --dport 8118 -m conntrack --ctstate NEW -j ACCEPT
iptables -t nat -A PREROUTING -i $DMZ -p tcp --dport 80 -j DNAT --to-destination 172.16.1.1:8118
iptables -A INPUT -i $WIFI -s $WIFI_NET -p tcp --dport 8118 -m conntrack --ctstate NEW -j ACCEPT
iptables -t nat -A PREROUTING -i $WIFI -p tcp --dport 80 -j DNAT --to-destination 192.168.1.1:8118

echo "Example of blocking by IP address"
iptables -A FORWARD -d 209.18.47.62 -j DROP
iptables -A FORWARD -d 209.18.47.61 -j DROP

echo "Allow outgoing to Internet"
iptables -A OUTPUT -o $EXT -d 0.0.0.0/0 -j ACCEPT

echo "Allow traffic from the firewall to LAN"
iptables -A OUTPUT -o $INT -d $INT_NET -j ACCEPT
iptables -A OUTPUT -o $DMZ -d $DMZ_NET -j ACCEPT
iptables -A OUTPUT -o $WIFI -d $WIFI_NET -j ACCEPT

echo "Enable NAT"
iptables -t nat -A POSTROUTING -o $EXT -j MASQUERADE
iptables -A FORWARD -o $EXT -i $INT -s $INT_NET -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -o $EXT -i $DMZ -s $DMZ_NET -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -o $EXT -i $WIFI -s $WIFI_NET -m conntrack --ctstate NEW -j ACCEPT

echo "Log dropped packets (to /var/log/kern.log)"
iptables -A INPUT -m limit --limit 1/sec -j LOG --log-level debug --log-prefix 'IN>'
iptables -A OUTPUT -m limit --limit 1/sec -j LOG --log-level debug --log-prefix 'OUT>'
iptables -A FORWARD -m limit --limit 1/sec -j LOG --log-level debug --log-prefix 'FWD>'
```

When you're finished, use `chmod +x iptables.sh` to make the file executable and apply it with `sudo ./iptables.sh`.

To make the firewall rules permanent *after* applying them:

    $ sudo iptables-save | sudo tee /etc/iptables/rules.v4

Add their restoration to `/etc/rc.local`, for example:

```
#!/bin/sh -e

# restore firewall rules
sudo /sbin/iptables-restore < /etc/iptables/rules.v4

exit 0
```

# Configure DNSCrypt

Install DNSCrypt client to encrypt outgoing DNS from https://download.dnscrypt.org/dnscrypt-proxy/

```
$ curl -O https://download.dnscrypt.org/dnscrypt-proxy/dnscrypt-proxy-1.7.0.tar.gz

$ curl -O https://download.dnscrypt.org/dnscrypt-proxy/dnscrypt-proxy-1.7.0.tar.gz.sig

$ gpg dnscrypt*sig
gpg: assuming signed data in `dnscrypt-proxy-1.7.0.tar.gz'
gpg: Signature made Sun 31 Jul 2016 07:14:47 AM EDT using RSA key ID 2B6F76DA
gpg: Can't check signature: public key not found

$ gpg --recv 0x2b6f76da
gpg: requesting key 2B6F76DA from hkp server keys.gnupg.net
gpg: key BA709FE1: public key "Frank Denis (Jedi/Sector One) <pgp@pureftpd.org>" imported
gpg: no ultimately trusted keys found
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)

$ gpg dnscrypt*sig
gpg: assuming signed data in `dnscrypt-proxy-1.7.0.tar.gz'
gpg: Signature made Sun 31 Jul 2016 07:14:47 AM EDT using RSA key ID 2B6F76DA
gpg: Good signature from "Frank Denis (Jedi/Sector One) <pgp@pureftpd.org>"
gpg:                 aka "Frank Denis <github@pureftpd.org>"
gpg:                 aka "Frank Denis <frank.denis@corp.ovh.com>"
gpg:                 aka "Frank Denis (Jedi/Sector One) <j@pureftpd.org>"
gpg:                 aka "Frank Denis (Jedi/Sector One) <0daydigest@pureftpd.org>"
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 54A2 B889 2CC3 D6A5 97B9  2B6C 2106 27AA BA70 9FE1
     Subkey fingerprint: 0C79 83A8 FD9A 104C 6231  72CB 62F2 5B59 2B6F 76DA

$ tar xf dnscrypt-proxy-1.7.0.tar.gz

$ sudo apt-get -y install libsodium-dev libevent-dev

$ ./configure

$ make -sj4

$ sudo make install
```

Start DNSCrypt:

```
$ sudo dnscrypt-proxy -a 127.0.0.1:40 -r 104.196.xxx.xxx:5355 -k 0FE2:[...]:13AC -N 2.dnscrypt-cert.duh.to
[NOTICE] Starting dnscrypt-proxy 1.7.0
[INFO] Generating a new session key pair
[INFO] Done
[INFO] Server certificate with serial #1475108830 received
[INFO] This certificate is valid
[INFO] Chosen certificate #1475108830 is valid from [2016-09-29] to [2017-09-29]
[INFO] Server key fingerprint is C8AE:[...]:56A2
[NOTICE] Proxying from 127.0.0.1:40 to 104.196.xxx.xxx:5355
```

To use DNSCrypt, edit `/etc/dnsmasq.conf` to include `server=127.0.0.1#40` and `sudo service dnsmasq restart`

Verify outgoing DNS is encrypted while quering a record, for example using `dig a google.com`:

```
$ sudo tcpdump -As800 -ni eth0 'tcp or udp'
listening on eth0, link-type EN10MB (Ethernet), capture size 800 bytes
IP 10.8.4.1.40259 > 104.196.xxx.xxx.5355: UDP, length 512
E....$..@..7D...h..Q.C..............;b..>..u.]@.'....|.|.>|.2q..Z:..,..%0y..*.f^. .,.....fv.| .....}...........-i;{.w..@1....T.E$jP.q.....CV.7....".3]...,|......u..< .............u<@..3..^..m.MWD...s...}.u..v-..!.(....n..o..a.x.{....b$.kB0..4w..U...Y<x.4....~....l.......Q..m...Knx|.W..EI...h.
#..ciQd..u..v..I`......Z$.gD..Gx....M.....!./f...d..-V...8$...bv.;..|.h..W~[.tj.R.....j.u..j..77.j.m..c.pF...:...cLu)_..}.UrH.YfuJ....bV...iK@n.lb....G.J...7.#.o.s...:.!....3.g..^..X../. .....2.
..th.....}.......{P,C?..O,f..^.w......:.h.....2HJ.{
14:00:07.161703 IP 104.196.xxx.xxx.5355 > 10.8.4.1.40259: UDP, length 304
E..L7s..7.R.h..QD......C.8].r6fnvWj8,..%0y..*.f^..y/.[...+.K..._.u..]..>JTe..~....zA......T..+.%....j....n..-#.b....'Q..8.......C.,.R...4.).p....C.$k.p....@G.@@_BT.*.......v];...s....6....p..|.=.H".#}...r{.9..';.L...   .;.b0j,.y_.Y=..+..O.....N/..e.......M.+2.....q"Gl..v.E.+.[...`.....J....1...*....-Lx...X....6e.B.......j..[.C{.^.&.
^C
```

Add DNSCrypt to `/etc/rc.local` so that it starts on boot, for example:

```
#!/bin/sh -e

# restore firewall rules
sudo /sbin/iptables-restore < /etc/iptables/rules.v4

# start dnscrypt-proxy (note the "&" to background the process)
sudo dnscrypt-proxy -a 127.0.0.1:40 -r 104.196.xxx.xxx:5355 -k 0FE2:[...]:13AC -N 2.dnscrypt-cert.duh.to &

exit 0
```

To run your own DNSCrypt server, see [drduh/Debian-Privacy-Server-Guide#dnscrypt](https://github.com/drduh/Debian-Privacy-Server-Guide#dnscrypt).

# Configure ad-blocking


# Ad-blocking

Install [Privoxy](https://www.privoxy.org/) and [Lighttpd](https://www.lighttpd.net/) with [mod_magnet](https://redmine.lighttpd.net/projects/1/wiki/Docs_ModMagnet)

    $ sudo apt-get install -y privoxy lighttpd lighttpd-mod-magnet

Edit default configurations, or download and edit mine:

```
$ curl https://raw.githubusercontent.com/drduh/config/master/lighttpd.conf | sudo tee /etc/lighttpd/lighttpd.conf

$ sudo vim /etc/lighttpd/lighttpd.conf

$ curl https://raw.githubusercontent.com/drduh/config/master/magnet.luau | sudo tee /etc/lighttpd/magnet.luau

$ sudo vim /etc/lighttpd/magnet.luau

$ curl https://github.com/drduh/config/blob/master/privoxy | sudo tee /etc/privoxy/config

$ sudo vim /etc/privoxy/config
```

Restart both services and check to make sure they work.

```
$ sudo service lighttpd restart

$ sudo service privoxy restart
```

To set an image blocker, edit `/etc/privoxy/user.action` to add the following to the bottom:

```
{+set-image-blocker{http://10.8.4.1/}}
/.*.[jpg|jpeg|gif|png|tif|tiff]$
```

# Conclusion

Congratulations, you've assembled, installed and configured a powerful wireless router for less than $200. Now you can stop wasting your money on off-the-shelf networking gear.

If you want to make sure you've set up your firewall correctly, run a port scan from an external host:

    $ nmap -v -A -T4 xxx.xxx.xxx.xxx -Pn

# Todo

* Configure 802.1x authentication for WLAN
* Implement on a BSD
* Sandbox/harden programs
* Automate software updates
* Email alerts for low disk space, etc.
