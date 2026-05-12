---
layout: post
title: "The mdadm backup file: when it's needed, and when it isn't"
description: "A close look at what mdadm's --backup-file does during a RAID reshape: when it's required, when it's optional, and what happens if you lose it mid-operation."
date: 2026-05-13 18:00:00 +0200
tags: [linux, storage, mdadm, raid]
---

I'm planning a home storage build, and one thing I want to understand before I commit is how to grow a RAID5 array safely. Here's what I've learned.

For demonstration purposes I'm going to do it in VirtualBox. You can say it’s bottlenecked intentionally for demonstration purposes, and it’s at least partially true.

This is how I'm going to reach my starting point from zero:

```
sudo parted /dev/sdb --script mklabel gpt mkpart primary 1MiB 100% set 1 raid on
```
the 1MiB partition alignment matters for RAID performance, make sure your partitions align to mebibyte boundary.
```
sudo parted /dev/sdc --script mklabel gpt mkpart primary 1MiB 100% set 1 raid on
sudo parted /dev/sdd --script mklabel gpt mkpart primary 1MiB 100% set 1 raid on
```
I'm doing this on VirtualBox, all the affected virtual hard drives are the same capacity
```
sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb1 /dev/sdc1 /dev/sdd1
sudo mkfs.ext4 /dev/md0
sudo mount /dev/md0 /mnt
sudo chown $USER:$USER /mnt
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf
echo '/dev/md0 /mnt ext4 defaults 0 0' | sudo tee -a /etc/fstab
dd if=/dev/urandom bs=1M count=150 of=/mnt/testfile
sha512sum /mnt/testfile >/mnt/testfile.sha512
```

All right. Now that I have my RAID array, and my precious data, and I need more space for more data, I'm going to expand my RAID array. Here's how I'm going to do it:

```
sudo mdadm --add /dev/md0 /dev/sde1
```
at this point the new hdd is added as a spare drive, here's how the current state can be checked:
```
sudo mdadm --detail /dev/md0
```
to actually grow I'm going to issue this command:
```
sudo mdadm --grow /dev/md0 --raid-devices=4
```
and then we can check progress by issuing:
```
cat /proc/mdstat
```
The fs was mounted and usable all along, however it didn't get resized, it will still report the old capacity (256MiB in my case):
```
df /mnt
```
this is how to resize it (works while still mounted as well):
```
sudo resize2fs /dev/md0
```
at this point `df /mnt` will report the capacity being increased.

## chunks and stripes explained
A chunk is the least amount of data considered by RAID. Chunks on the disks are aligned at multiples of the chunk size. A stripe is all the data represented by the chunks at the same offset of all the participating drives.
## reshaping: under the hood
first of all, let's remember: we are reshaping a RAID array while the underlying fs is still mounted, applications can still read/write and the operation happening below should be invisible (apart from latency). Now. When we're in the middle of the reshaping there will be a portion of the array where the reshaping has already happened, and therefore can be read/written with the new geometry in mind. This area is before `suspend_lo` as per the [official documentation](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/md.rst). Also, there's a portion which hasn't been reshaped yet, and this portion of the disk (above `suspend_hi`) is safe to be read/written as per the old geometry.

If the array is accessed in the in-between range, the operation will be blocked until the reshaping is done for the relevant portion of the array. All that will cause is some latency in IO. So overall, you're free to use your array during the reshaping. Technically there's a third value at play here as well: `reshape position` telling exactly up until which point in the array the new geometry should be considered. This value obviously cannot be outside the range discussed earlier. (In case of RAID0 slightly different mechanics are used, meaning the suspend_ values are omitted completely.)

The aforementioned three values can be observed reading the appropriate file, e.g. `/sys/block/md0/md/suspend_lo`

So we're happy users of a RAID array currently being reshaped, and confidently so. As we move forward in time, on the old disks the gap between the ready to use new region and the yet to be reshaped region widens, the space between is fair game for anything that needs to be stored temporarily.

But early during the reshaping process chunks to be written and chunks to be read were overlapping. In this critical section of the reshaping process it's essential to have some space for backup data, it either goes into a separate file, or into the newly added disk, therefore backup files were invented. Which are not always files, but anyhow we want to store some backup data because of the possibility of a system crash and because we don't want to use a considerable portion of the IO bandwidth to update metadata.

In essence early during reshape in case of a system crash there would be an uncertainty whether we have the old data in place, or the new one. This uncertainty is cleared up by the backup file: in case of a system crash we need to be able to restore to a known state and resume reshaping from there. That's what the backup data is for.

But in the scenario I'm in something even more clever happened: this is what I've seen mid-reshape:
```
$ sudo mdadm --examine /dev/sdb1
[...]
    Data Offset : 4096 sectors
     New Offset : 1024 sectors
[...]
```
and all the disks reported the same. Meaning the actual data got shifted by 1.5MiB on all disks, meaning even early during RAID reshaping there was no critical section in the sense there was no portion of the RAID array that should have been claimed by both the old and new geometry at the same time.

After this first grow, the data offset had shifted from 4096 to 1024 sectors. There wasn't enough remaining headroom at the start of the disks to do the same trick again. So when I added another drive to see what would happen, `mdadm` said `Need to backup 6144K of critical section`. There was no backup file necessary, after all, we have a completely empty drive at hand, so no surprises here.

It's worth noting that adding a new drive like this increases the probability of a drive failure just because there are more disks.

### sync_speed_max
One small thing I stumbled upon during writing this article is the following:
```
echo 10 | sudo tee /sys/block/md0/md/sync_speed_max
```
this limits the syncing to 10 kibibytes per second. Choosing such a small number only makes sense if you want to have enough time to inspect a small array during reshaping.

## reshaping from RAID5 to RAID6
Losing two disks is less probable than losing one, and RAID6 protects against two disk failures, so now we're reshaping from RAID5 to RAID6 by adding a new drive. So we're going from RAID5 with four drives to RAID6 with five drives, more redundancy, usable capacity won't change:
```
sudo parted /dev/sdf --script mklabel gpt mkpart primary 1MiB 100% set 1 raid on
sudo mdadm --add /dev/md0 /dev/sdf1
sudo mdadm --grow /dev/md0 --level=6 --raid-devices=5
```
ok, so I was surprised that it works like this and does not require a separate backup file, however it makes sense: I'm adding a new currently empty drive, also, as we've seen, data offset can give us some free space here or there as well, this is what I've seen mid-reshape:
```
    Data Offset : 1024 sectors
     New Offset : 2048 sectors
```
the unused space before (or after) the real data could be used for the reshaping as well.
But I'm not satisfied, I want to see the backup file, hopefully changing the chunk size would require me to use a backup file, let's try it:
```
sudo mdadm --grow /dev/md0 --chunk=256 --backup-file=/root/md0-reshape.bak
```
the backup file contains a safe backup/resume point from where the system is able to continue reshaping. That means: the file should be available after reboot without `md0` being online. My `/root` directory here survives reboots, and is outside `md0`, so it's a safe place for the backup.
when examining the drives you'll see something like this:
```
  Reshape pos'n : 39936 (39.00 MiB 40.89 MB)
  New Chunksize : 256K
```
after the reshape is done, you'll see:
```
         Layout : left-symmetric
     Chunk Size : 256K
```
ok so I wanted to see an error message saying the backup file is required, so I issued
```
sudo mdadm --grow /dev/md0 --chunk=128
```
which was surprisingly accepted, however was very slow compared to the previous one, because of the lack of the backup file.

I redid the whole experiment with and without a backup file. Without one, the reshape stalled while online, but completed offline. `mdadm --assemble --scan` took ages, the array was unavailable during that period, but `mdstat` reported progress. Even so, the no-backup-file path was significantly slower than the with-backup-file path.

But I still want to push a bit further, maybe removing a device will make it require a backup file (in theory if you remove a drive, that means the corresponding space at the end of md0 is empty, meaning that the space at the end of the removed drive corresponding to the empty portion of the md0 would be free to use. I'm going to figure out what happens as I write this)
```
sudo umount /mnt
sudo e2fsck -f /dev/md0
sudo resize2fs -M /dev/md0
```
now at this step the ext2 fs was larger than the theoretical capacity of the reduced array, which I don't understand, given we started with 2+1 drives, and we're reducing to 2+2 drives, maybe resize2fs made metadata to grow, and now it doesn't want to shrink it back, but anyhow I just recreated the test data with smaller size
```
$ sudo mdadm --grow /dev/md0 --raid-devices=4 --backup-file=/root/md0-reshape.bak
mdadm: this change will reduce the size of the array.
       use --grow --array-size first to truncate array.
       e.g. mdadm --grow /dev/md0 --array-size 254976
$ sudo mdadm --grow /dev/md0 --array-size 254976
$ sudo mdadm --grow /dev/md0 --raid-devices=4
mdadm: failed to write '4608' to '/sys/block/md0/md/dev-sdf1/new_offset' (Invalid argument)
mdadm: Cannot set new_offset for /dev/sdf1
```
the error message is not particularly user friendly, however reflects the internal working: mdadm tries to make room by making an offset of 4608, which then deemed invalid. Anyhow, the issue can be fixed by providing a backup file:
```
$ sudo mdadm --grow /dev/md0 --raid-devices=4 --backup-file=/root/md0-reshape.bak
mdadm: Need to backup 768K of critical section..
```
so in this case we know for sure the backup file is used this time, and we couldn't have gotten away without it. The backup file is safe to delete when the reshaping is done (as reported by `cat /proc/mdstat`).

## things I'd do differently in production
Given that I worked with virtual HDDs there was no point in doing SMART tests, but with real physical hardware it would be wise to do a full SMART test on each drive before doing anything. Also, given the fact that reshaping took place in minutes after creation, I was confident that the array has no latent issues yet. In production, you'd rarely reshape an array minutes after its creation. I'd want to scrub after the SMART tests are done. If there's anything surfacing during SMART tests or scrubbing, the array is still in a known good state, much easier to deal with it now than during (or after) reshaping.
# What did we learn?
We learned that in modern linux the backup file is superfluous in most of the cases. We've also seen a case where the lack of backup file was accepted, but reshaping stalled when the array was online. Therefore we learned that even if it might be unnecessary, having that file won't hurt. We learned that mdadm in some cases is able to do some clever tricks to avoid the necessity of backing up at all, in other cases the backup data is hidden away in unused portions of the underlying partitions. And we've seen that during shrinking the presence of a backup file is strictly necessary. Overall we've seen how mdadm makes reshape crash-safe across a variety of cases, sometimes via the backup file, sometimes by cleverer mechanisms that avoid needing one.
