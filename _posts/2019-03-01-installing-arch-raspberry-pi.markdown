---
layout: post
title:  "Installing Arch on a Raspberry Pi"
date:   2019-03-01 09:19:33
categories: arch blackarch raspberrypi tutorials
---

*Note: This post from 2016 has been moved from a previous blog ran with my frequent partner in crime [Jon Cornwell](http://joncornwell.com)*

Who doesn’t love the Raspberry Pi? Affordable, well supported and widely available, I’ve used it for everything from VPN servers to hardware projects.  I’m also a big fan of Arch linux, whose lightweight profile and deep community support make it a natural fit for the device.

The official Arch Linux wiki page for the Raspberry Pi has all the information you need to get started, however the instructions can be a little dense. We’ll be setting up Arch on a Raspberry Pi 2, but the process is essentially the same for other models of the device, you’ll just need to be sure to use the Arch image that corresponds to your hardware. 

## Preparing the SD card

The first thing we need to do is format the SD card and create a couple new partitions for our boot sector and root mount point. We’ll need to know which block device has been assigned to the card, so be sure to watch the system log as you plug your SD card into the reader. 

{% highlight bash %}
$ journalctl -f
#=> Dec 20 12:28:26 wreet kernel: usb 1-9: new high-speed USB device number 9 using xhci_hcd
#=> Dec 20 12:28:26 wreet kernel: usb-storage 1-9:1.0: USB Mass Storage device detected
#=> Dec 20 12:28:26 wreet kernel: scsi host7: usb-storage 1-9:1.0
#=> Dec 20 12:28:27 wreet kernel: scsi 7:0:0:0: Direct-Access     Generic- SD/MMC           1.00 PQ: 0 ANSI: 0 CCS
#=> Dec 20 12:28:27 wreet kernel: sd 7:0:0:0: [sdb] Sense not available.
#=> Dec 20 12:28:27 wreet kernel: sd 7:0:0:0: [sdb] Write Protect is off
#=> Dec 20 12:28:27 wreet kernel: sd 7:0:0:0: [sdb] Mode Sense: 00 00 00 00
#=> Dec 20 12:28:27 wreet kernel: sd 7:0:0:0: [sdb] Asking for cache data failed
#=> Dec 20 12:28:27 wreet kernel: sd 7:0:0:0: [sdb] Assuming drive cache: write through
#=> Dec 20 12:28:27 wreet kernel: sd 7:0:0:0: [sdb] Attached SCSI removable disk
{% endhighlight %}

In this case, “sdb” is the relevant device. Once you know the correct block device, it’s time to format the card in preparation for the Arch image. We’ll use the fdisk tool to clear the existing partition table and write a new one that the Raspberry Pi can boot from. Please note this operation will permanently erase all information on the disk. 
{% highlight bash %}
$ sudo fdisk /dev/sdb
#=> Welcome to fdisk (util-linux 2.27.1).
#=> Changes will remain in memory only, until you decide to write them.
#=> Be careful before using the write command.
#=> Command (m for help):
{% endhighlight %}

fdisk steps:
Type **o** then enter to clear the existing table. 

{% highlight bash %}
#=> Command (m for help): o
#=> Created a new DOS disklabel with disk identifier [id]
{% endhighlight %}

Hit **n** for a new partition, then **p** for primary. Next, press enter to accept the default partition number, which should be 1. Hit enter to accept the default start sector, then **+100M** as the last sector. This will create a 100MB partition that will serve as the boot disk. 

{% highlight bash %}
#=> Command (m for help): n
#=> Partition type
#=>   p   primary (0 primary, 0 extended, 4 free)
#=>   e   extended (container for logical partitions)
#=> Select (default p): p
#=> Partition number (1-4, default 1):  
#=> First sector (2048-62521343, default 2048): 
#=> Last sector, +sectors or +size{K,M,G,T,P} (2048-62521343, default 62521343): +100M
#=> Created a new partition 1 of type 'Linux' and of size 100 MiB.
{% endhighlight %}

In order to boot, the Raspberry Pi expects this sector to contain a FAT filesystem. Type **t** and then **c** to change the partition type. 

{% highlight bash %}
#=> Command (m for help): t
#=> Selected partition 1
#=> Partition type (type L to list all types): c
#=> Changed type of partition 'Linux' to 'W95 FAT32 (LBA)'.
{% endhighlight %} 

For the root fs mount point, we’ll create another partition as before with fdisk. Type **n** and then **p**, and hit enter to accept the default partition number of 2. Press the enter key twice more to accept the default start and end sectors, which will include every free sector that comes after our new boot partition -- all the remaining space on the drive. If you would like to further partition the disk, for example to add a swap partition, in the “last sector” prompt simply specify how much space the root fs should take. For example, if I wanted to leave 2GB for a swap partition, I’d put “+27.7G” when prompted.

{% highlight bash %}
#=> Command (m for help): n
#=> Partition type
#=>   p   primary (1 primary, 0 extended, 3 free)
#=>   e   extended (container for logical partitions)
#=> Select (default p): p
#=> Partition number (2-4, default 2): 
#=> First sector (206848-62521343, default 206848): 
#=> Last sector, +sectors or +size{K,M,G,T,P} (206848-62521343, default 62521343): 
#=> Created a new partition 2 of type 'Linux' and of size 29.7 GiB.
{% endhighlight %}

Write the new table by hitting **w**. After finishing the changes fdisk will exit automatically.

Next, we will write the actually filesystems to our new partitions. We will start with the boot partition. For convenience, I typically just create a new folder in my current working directory for each partition to serve as their mount points. 

*Note: mkfs.vfat may require an additional package, such as ‘dosfstools’ on Arch*

{% highlight bash %}
$ mkdir boot root
$ sudo mkfs.vfat /dev/sdb1
#=> mke2fs 1.42.13 (17-May-2015)
$ sudo mkfs.ext4 /dev/sdb2
#=> mke2fs 1.42.13 (17-May-2015)
#=> Creating filesystem with 7789312 4k blocks and 1949696 inodes
#=> Filesystem UUID: eeb394b6-0570-4dbe-8798-436276711342
#=> Superblock backups stored on blocks: 
#=>    32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
#=>    4096000
#=> Allocating group tables: done                            
#=> Writing inode tables: done                            
#=> Creating journal (32768 blocks): done
#=> Writing superblocks and filesystem accounting information: done
$ sudo mount /dev/sdb1 boot
$ sudo mount /dev/sdb2 root
{% endhighlight %}

## Installing the Arch image

Finally, we are ready to download our image and extract it onto the SD card. Download the appropriate image with wget, and extract it into the root directory. 

*Note: be sure to wget the correct image for your hardware.*
{% highlight bash %}
$ wget http://archlinuxarm.org/os/ArchLinuxARM-rpi-2-latest.tar.gz
$ tar zxvfp ArchLinuxARM-rpi-2-latest.tar.gz -C root/
$ sudo sync
{% endhighlight %}

One of the newly extracted folders in the root directory will be labeled ‘boot’ and contains all the files that will be required in the boot partition. Simply move them to their new location. 

{% highlight bash %}
$ sudo mv root/boot/* boot/
{% endhighlight %}

Now unmount the disks.

{% highlight bash %}
$ sudo umount boot root
{% endhighlight %}

Congratulations, your SD card should now be loaded with Arch and ready to use with your Raspberry Pi! By default, Arch will run a DHCP client for the ethernet port and accept SSH connections. Login with the username *alarm* and password *alarm*. The default *root* password is *root*. 

## Bonus: Black Arch

For the majority of my Arch setups, I like to add the Black Arch repository to my pacman config. That way I can easily install any security or pentesting tools I’d like to use without any hassle. The instructions can be found [here](https://blackarch.org/downloads.html) on the Black Arch website, but it is essentially as simple as running a shell script. 

{% highlight bash %}
$ curl -O https://blackarch.org/strap.sh
{% endhighlight %}

Verify the sha1 sum of the script, at the time of writing it is `9f770789df3b7803105e5fbc19212889674cd503`, but you should always check the Black Arch website for the latest sum. Since running scripts from the internet is dangerous, especially as root, I would recommend reading the content of the script. It’s not too long, and the actions being taken are pretty simple. 

Make the script executable, and run it as root.

{% highlight bash %}
$ chmod +x strap.sh
$ sudo ./strap.sh
{% endhighlight %}

You will now be able to pull any tool provided by the Black Arch repo using pacman. A full list of provided packages can be found [here](https://blackarch.org/tools.html). 
