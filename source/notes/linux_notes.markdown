---
layout: page
title: "Linux Tips"
date: 2013-05-26 19:50
comments: true
sharing: true
footer: true
---

Some tips about Linux:

------

#### mount *.img to /mnt

	$ fdisk -l *.img

	Disk guest-16G.img: 17.2 GB, 17179869184 bytes
	255 heads, 63 sectors/track, 2088 cylinders, total 33554432 sectors
	Units = sectors of 1 * 512 = 512 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 512 bytes
	Disk identifier: 0x0008e666

    	    Device Boot      Start         End      Blocks   Id  System
	guest-16G.img1   *        2048    32088063    16043008   83  Linux
	guest-16G.img2        32090110    33552383      731137    5  Extended
	guest-16G.img5        32090112    33552383      731136   82  Linux swap / Solaris
 
	$ mount -o offset=$((2048*512)),loop guest-16G.img /mnt

------

#### lower the FSCK frequency

We can run tune2fs on the ext3 partition with the ‘-l‘ option to view how many times filesystem has been mounted after creation and number of mounts after which the filesystem will be checked:

	$ sudo tune2fs -l /dev/sda1

To turn off this check set the maximum count to 0 with the ‘-c‘ option.

	$ sudo tune2fs -c 0 /dev/sda1

If you do not want to completely disable the file system checking, you can also increase the maximum count.

	$ sudo tune2fs -c 100 /dev/sda1