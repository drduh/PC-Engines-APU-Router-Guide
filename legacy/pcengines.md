> [!IMPORTANT]
> After many years of service, the PC Engines APU platform is now [EOL](https://www.pcengines.ch/eol.htm).

## Hardware

Part | Description | Cost
-: | :-: | :-
[apu4c4](https://pcengines.ch/apu4c4.htm) | apu4c4 system board | $117.50
[case1d2bluu](https://pcengines.ch/case1d2bluu.htm) | Enclosure 3 LAN, blue | $9.40
[ac12vus2](https://pcengines.ch/ac12vus2.htm) | AC adapter 12V 2A US plug | $4.10
[msata16g](https://pcengines.ch/msata16g.htm) | SSD M-Sata 16GB MLC, Phison S11 | $15.50
[wle200nx](https://pcengines.ch/wle200nx.htm) | Compex WLE200NX miniPCI express card | $19.00
[pigsma](https://pcengines.ch/pigsma.htm) (x2) | Cable I-PEX -> reverse SMA | $2.70
[antsmadb](https://pcengines.ch/antsmadb.htm) (x2) | Antenna reverse SMA dual band | $4.10

> [!NOTE]
> WLE600VX and WLE900VX cards will likely not work due to [regulatory compliance reasons](https://medium.com/@renaudcerrato/how-to-build-your-own-wireless-router-from-scratch-part-3-d54eecce157f).

To connect over serial, you will need a [USB to Serial (9-Pin) Converter Cable](https://www.amazon.com/gp/product/B00IDSM6BW) and [Modem Serial RS232 Cable](https://www.amazon.com/gp/product/B000067SCH), also available from [PC Engines](https://www.pcengines.ch/usbcom1a.htm).

See [Issue #1](https://github.com/drduh/PC-Engines-APU-Router-Guide/issues/1) for a list of alternative parts.

## Assembly

Clear a workspace and unpack the materials. Follow the [apu cooling assembly instructions](https://www.pcengines.ch/apucool.htm) to install the heat conduction plate.

Install the mSATA drive and miniPCIe wireless adapter in their respective slots.

See the relevant APU series manual for detailed board information:

* [APU2](https://www.pcengines.ch/pdf/apu2.pdf)
* [APU3](https://www.pcengines.ch/pdf/apu3.pdf)
* [APU4](https://www.pcengines.ch/pdf/apu4.pdf)

> [!CAUTION]
> Wireless radio cards are ESD sensitive, especially the RF switch and the power amplifier. To avoid damage by electrostatic discharge, the following installation procedure is [recommended](https://www.pcengines.ch/wle200nx.htm)

1. Touch your hands and the bag containing the radio card to a ground point on the router board (for example one of the mounting holes). This equalizes the electrical potential between the radio card and the router board.
1. Install the radio card in the miniPCI express socket.
1. Install the pigtail cable in the cut-out of the enclosure. This will ground the pigtail to the enclosure.
1. Touch the pigtail's I-PEX connector to a mounting hole to discharge it, then plug it into the radio card.

To avoid arcing, connect the DC jack first, then plug the power adapter into an outlet.

Press `F10` during boot and select `Payload [memtest]` to complete at least one pass.

# Connect over serial

The APU serial connection uses 115200 baud, 8N1 (8 data bits, no parity, and 1 stop bit).

On OpenBSD, use [cu](https://man.openbsd.org/cu):

```bash
doas cu -r -s 115200 -l cuaU0
```

On Linux, use [screen](https://www.gnu.org/software/screen/manual/screen.html):

```bash
screen /dev/ttyUSB0 115200 8N1
```

Or use [minicom](https://linux.die.net/man/1/minicom):

```bash
sudo minicom -D /dev/ttyUSB0
```

Power on the APU and note the firmware version displayed briefly during boot.

# Updating firmware

Check for the latest PC Engines firmware version at [pcengines.github.io](https://pcengines.github.io/)

> [!NOTE]
> As of 2023, PC Engines firmware is no longer being updated. See [announcement](https://docs.dasharo.com/variants/pc_engines/post-eol-fw-announcement/).

To update firmware, first download and extract [TinyCore Linux](https://pcengines.ch/file/apu2-tinycore6.4.img.gz).

Download and import the [firmware signing key](https://github.com/3mdeb/3mdeb-secpack/tree/master/customer-keys/pcengines/release-keys), then check the file signature:

```console
$ curl -LO https://raw.githubusercontent.com/3mdeb/3mdeb-secpack/master/customer-keys/pcengines/release-keys/pcengines-open-source-firmware-release-4.19-key.asc

$ gpg --import pcengines-open-source-firmware-release-4.19-key.asc
gpg: key 0x30A53DE2F5A6D89A: 1 signature not checked due to a missing key
gpg: key 0x30A53DE2F5A6D89A: public key "PC Engines open-source firmware release 4.19 signing key" imported
gpg: Total number processed: 1
gpg:               imported: 1

$ gpg apu4_v4.19.0.1.SHA256.sig
gpg: assuming signed data in 'apu4_v4.19.0.1.SHA256'
gpg: Signature made Thu 02 Feb 2023 03:22:57 AM PST
gpg:                using RSA key 05CF36F166C3D676A08AB70F30A53DE2F5A6D89A
gpg: Good signature from "PC Engines open-source firmware release 4.19 signing key" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 05CF 36F1 66C3 D676 A08A  B70F 30A5 3DE2 F5A6 D89A

$ shasum -a 256 apu4_v4.19.0.1.rom 2>/dev/null | grep -q $(cat apu4_v4.19.0.1.SHA256 | awk '{print $1}') && echo ok
ok
```

Mount a USB disk and write the TinyCore image, copy the `.rom` file:

```bash
curl -O https://pcengines.ch/file/apu2-tinycore6.4.img.gz

gzip -d apu2-tinycore6.4.img.gz

sha256sum apu2-tinycore6.4.img
f5a20eeb01dfea438836e48cb15a18c5780194fed6bf21564fc7c894a1ac06d7  apu2-tinycore6.4.img

sudo dd if=apu2-tinycore6.4.img of=/dev/sdd bs=1M

sudo mkdir /mnt/usb

sudo mount /dev/sdd1 /mnt/usb

sudo cp -v apu4_*.rom /mnt/usb

sudo umount /mnt/usb
```

Connect the USB disk to the APU. During boot, press `F10` and select the USB disk.

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

root@pcengines:/media/SYSLINUX# flashrom -p internal -w apu4_v4.19.0.1.rom
[...]
Found Winbond flash chip "W25Q64.V" (8192 kB, SPI) mapped at physical address 0xff800000.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
```

Unplug the USB disk and reboot.

**Optional** On reboot, press `F10`, select `Payload [setup]`, press `w` to enable BIOS write protection, then press `s` to save and reboot.

Verify the version by checking serial output during boot:

```
PC Engines apu4
coreboot build 20230131
BIOS version v4.19.0.1
```

From OpenBSD:

```console
$ dmesg | grep bios
bios0 at mainbus0: SMBIOS rev. 2.8 @ 0xcfe8b020 (13 entries)
bios0: vendor coreboot version "v4.19.0.1" date 01/31/2023
bios0: PC Engines apu4
acpi0 at bios0: ACPI 6.0
```

From Debian:

```console
$ sudo dmesg | grep apu
[    0.000000] DMI: PC Engines apu4/apu4, BIOS v4.19.0.1 01/31/2023
```

APU firmware can also be updated from Debian without rebooting to TinyCore Linux:

```bash
sudo apt install flashrom
wget https://3mdeb.com/open-source-firmware/pcengines/apu4/apu4_v4.19.0.1.rom
sudo flashrom -p internal -w apu4_v4.19.0.1.rom
```

To complete the update, shut down Debian and power off the APU fully, then reboot.
