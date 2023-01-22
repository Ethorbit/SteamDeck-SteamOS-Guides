# Steam Deck on SteamOS with LUKS Encryption
This is a Steam Deck SteamOS guide for LUKS encryption which works with the on-screen keyboard and does not require a SteamOS reinstall. 

To make this all possible; Full Disk Encryption simply can't work. We will be encrypting the SD card and /home instead, which will still protect your Steam logins, browser cookies, cache and other important user data. 

If you have any other custom partitions which SteamOS does not rely on to run, you can encrypt those as well.

# ![warning-icon](https://i.imgur.com/ZWdfbEN.png) Warning
**None of this is official**. I am not a developer for Valve. The method used here is very hacky, intended only for SteamOS and may not work in future releases. 

I'm not responsible for any damage or data loss.

# Prerequisites
* A **complete backup of all data stored on your SD card, system /home/ directory and anything else you decide to encrypt.** It's all getting erased.
* A Throwaway Steam account which will only be used to decrypt the disks (this is for best SteamOS compatibility, trust me)

* A USB-C hub with USB ports which work in the Steam Deck, preferably with power pass-through support so that you can charge the deck as well (securely wiping disks takes a long time)
* A USB flash drive with a Linux install(er) on it like [this](https://archlinux.org/download/), so you can access the SteamOS partitions from outside

(While you technically could just do everything inside SteamOS; having an installer and hub will ensure that if a fuckup happens which prevents booting, you'll have full ability to restore the deck back to a working state. It will also ensure that SteamOS will not get in your way during encryption.)

![Example](https://i.imgur.com/W7EdVYn.png)

# Enabling TPM and dm_crypt

For some reason, Valve has intentionally disabled TPM and dm_crypt. These are needed for encryption.

* Press the Steam button > Power and Switch to Desktop. 
* Find and open Konsole. 

If you haven't already setup a password: type `passwd` to do so.

Enter these commands:
* `echo "dm_crypt" | sudo tee /etc/modules-load.d/dm_crypt.conf`
* `sudo sed -i "s/module_blacklist=tpm//" /etc/default/grub`
* `sudo update-grub`

# Booting into a Linux installer
Despite its name, we are not actually going to be "installing" anything; we just need a separate temporary Linux environment to use while we manage and encrypt partitions. We are doing it this way to simply avoid any file conflicts. My flash drive has Ventoy with an Arch Linux Installer .iso on it, so I will be booting from that.

With the Deck powered off and usb devices plugged in, hold the Volume Down and Power button at the same time to reach the boot menu. Select the Flash drive

[![Video Tutorial](https://i.imgur.com/6VH6ZiY.jpg)](https://www.youtube.com/watch?v=2_Pkv4tr8Ho)


# Partitioning
To identify storage devices, use lsblk.

```
root@archiso ~ # lsblk

NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0 689.8M  1 loop /run/archiso/airootfs
sda           8:0    1     0B  0 disk 
sdb           8:16   1  57.8G  0 disk 
├─sdb1        8:17   1  57.7G  0 part 
│ └─ventoy  254:0    0 795.3M  1 dm   /run/archiso/bootmnt
└─sdb2        8:18   1    32M  0 part 
mmcblk0     179:0    0 477.5G  0 disk 
└─mmcblk0p1 179:1    0 477.5G  0 part 
nvme0n1     259:0    0  57.6G  0 disk 
├─nvme0n1p1 259:1    0    64M  0 part 
├─nvme0n1p2 259:2    0    32M  0 part 
├─nvme0n1p3 259:3    0    32M  0 part 
├─nvme0n1p4 259:4    0     5G  0 part 
├─nvme0n1p5 259:5    0     5G  0 part 
├─nvme0n1p6 259:6    0   256M  0 part 
├─nvme0n1p7 259:7    0   256M  0 part 
└─nvme0n1p8 259:8    0    47G  0 part 
```

Here, *nvme0n1* is the Steam Deck's internal storage and *mmcblk0* is the SD card.
Your numbers *may* be different, so keep an eye on that.

We will use fdisk <disk name, not partition name> (in my case that would be /dev/nvme0n1 or /dev/mmcblk0)

Note: *If you make a mistake in fdisk, Ctrl + C will exit without changes as long as you didn't enter w.*

### SD card
* `fdisk /dev/mmcblk0`
* Enter d and press enter and repeat until there are no more partitions left.
* Enter g to create a new GPT label
* Enter n to create a new partition
* Press enter until it stops asking (this means we're allocating the entire SD card with a single partition, if you're okay with that.)
* Enter w to write changes
  
### Home partition
![warning-icon](https://i.imgur.com/ZWdfbEN.png) We have to be careful here, because SteamOS is installed on the nvme drive!
  
We only care about the **home** partition of the nvme drive, which is the largest and last partition.

We can check partitions by doing fdisk -l <device name>
```
root@archiso ~ # fdisk -l /dev/nvme0n1
Disk /dev/nvme0n1: 57.62 GiB, 61865982976 bytes, 120831998 sectors
Disk model: E2M2 64GB
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: F77CFC1B-79AD-544C-B019-BDA1C67C0071

Device            Start       End  Sectors  Size Type
/dev/nvme0n1p1     2048    133119   131072   64M EFI System
/dev/nvme0n1p2   133120    198655    65536   32M Microsoft basic data
/dev/nvme0n1p3   198656    264191    65536   32M Microsoft basic data
/dev/nvme0n1p4   264192  10749951 10485760    5G Linux root (x86-64)
/dev/nvme0n1p5 10749952  21235711 10485760    5G Linux root (x86-64)
/dev/nvme0n1p6 21235712  21759999   524288  256M Linux variable data
/dev/nvme0n1p7 21760000  22284287   524288  256M Linux variable data
/dev/nvme0n1p8 22284288 120831964 98547677   47G Linux home
```
  
Here in my case, we can see the Type "Linux Home" is partition # 8 (the largest and last).
  
* `fdisk /dev/nvme0n1` 
* Enter d and then enter the # of the home partition
* Enter n, press enter twice until it asks for "Last Sector"
* For Last Sector, I will enter `-2G`. -2G means I'm leaving 2 gigabytes of space for the unencrypted partition we'll create next, which will only need enough for a Steam install. You can reduce it by more if you think it might require more space in the future.
* Enter Y to remove the signature (if it asks)
* Enter n and press enter until it stops asking. This will be unencrypted home.
* Enter p to check if your partitions look good. 
```
  Device             Start       End  Sectors  Size Type
/dev/nvme0n1p1      2048    133119   131072   64M EFI System
/dev/nvme0n1p2    133120    198655    65536   32M Microsoft basic data
/dev/nvme0n1p3    198656    264191    65536   32M Microsoft basic data
/dev/nvme0n1p4    264192  10749951 10485760    5G Linux root (x86-64)
/dev/nvme0n1p5  10749952  21235711 10485760    5G Linux root (x86-64)
/dev/nvme0n1p6  21235712  21759999   524288  256M Linux variable data
/dev/nvme0n1p7  21760000  22284287   524288  256M Linux variable data
/dev/nvme0n1p8  22284288 116637695 94353408   45G Linux filesystem
/dev/nvme0n1p9 116637696 120829951  4192256    2G Linux filesystem
```
  
For me, 8 has 45G and 9 has 2G. Perfect!
* Enter w to write changes
  
# Encrypting
My /dev/mmcblk0p1 and /dev/nvme0n1p8 are the two partitions that need to be encrypted.
  
![warning-icon](https://i.imgur.com/ZWdfbEN.png) Make sure you double check that the partitions you are about to encrypt are the ones **you** created, these next actions have the potential to bork your whole SteamOS install (if passed the wrong partition(s).)

We will use cryptsetup luksFormat to setup encryption for them. 
* `cryptsetup luksFormat /dev/mmcblk0p1`
* `cryptsetup luksFormat /dev/nvme0n1p8`

It will ask you to confirm YES and then to enter a secure, memorable password. If you forget this password, you're screwed. Keep backups of important files.
  
Next we will open them
* `cryptsetup luksOpen /dev/mmcblk0p1 crypt_sdcard`
* `cryptsetup luksOpen /dev/nvme0n1p8 crypt_home`

We should never use them directly as they are now encrypted. The decrypted (opened) devices are located at /dev/mapper/crypt_sdcard and /dev/mapper/crypt_home.
  
The next thing we will do is securely wipe them. **This is a process that will take a long time to complete**, especially with the slower SD card - which is why you should be charging your deck by now.

* `dd if=/dev/zero of=/dev/mapper/crypt_sdcard bs=1M status=progress`
* `dd if=/dev/zero of=/dev/mapper/crypt_home bs=1M status=progress`
  
Normally, this operation would allocate the entire disk with zeroes, but since all writes to these devices are encrypted, these zeroes will also be encrypted. This makes it way harder for attackers to figure out which parts of the encrypted disks actually contain user data.
  
# Mounting SteamOS /var partition

So SteamOS's /etc has two directories' contents merged - one of them is read-only and the other is read-write and located on the var partition's lib/overlays/etc/upper/. Therefore, in order to make changes to the /etc directory, we need to first mount the SteamOS /var partition.
 
```
root@archiso ~ # lsblk     
NAME           MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
loop0            7:0    0 689.8M  1 loop  /run/archiso/airootfs
sda              8:0    1     0B  0 disk  
sdb              8:16   1  57.8G  0 disk  
├─sdb1           8:17   1  57.7G  0 part  
│ └─ventoy     254:0    0 795.3M  1 dm    /run/archiso/bootmnt
└─sdb2           8:18   1    32M  0 part  
mmcblk0        179:0    0 477.5G  0 disk  
└─mmcblk0p1    179:1    0 477.5G  0 part  
nvme0n1        259:0    0  57.6G  0 disk  
├─nvme0n1p1    259:9    0    64M  0 part  
├─nvme0n1p2    259:10   0    32M  0 part  
├─nvme0n1p3    259:11   0    32M  0 part  
├─nvme0n1p4    259:12   0     5G  0 part  
├─nvme0n1p5    259:13   0     5G  0 part  
├─nvme0n1p6    259:14   0   256M  0 part  
├─nvme0n1p7    259:15   0   256M  0 part  
├─nvme0n1p8    259:16   0    45G  0 part  
│ └─crypt_home 254:1    0    45G  0 crypt 
└─nvme0n1p9    259:17   0     2G  0 part    
```  
  
As you can see, there are are two SteamOS nvme partitions with the size of 256M, you want to mount the first one.
  
`mount /dev/nvme0n1p6 /mnt`
  
with ls, you can check if the /mnt mount contains the right files:
```
root@archiso ~ # ls -lah /mnt
total 28K
drwxr-xr-x 17 root root  1.0K Jan 22 01:08 .
drwxr-xr-x  1 root root   100 Jan 22 05:21 ..
-rw-r--r--  1 root root   208 Dec 28 23:53 .updated
drwxr-xr-x  7 root root  1.0K Jan 10 19:03 cache
drwx--x--x  3 root root  1.0K Jan 15 19:12 db
drwxr-xr-x  2 root root  1.0K Jan 10 17:08 empty
drwxrwxr-x  2 root games 1.0K Jan 10 17:08 games
drwxr-xr-x 24 root root  1.0K Jan 10 19:03 lib
drwxr-xr-x  2 root root  1.0K Jan 10 17:08 local
lrwxrwxrwx  1 root root    11 Jan 10 17:08 lock -> ../run/lock
drwxr-xr-x  2 root root  1.0K Jan 10 17:08 log
drwx------  2 root root   12K Jan 10 17:19 lost+found
lrwxrwxrwx  1 root root    10 Jan 10 17:08 mail -> spool/mail
drwxr-xr-x  6 root root  1.0K Jan 21 13:21 mnt
drwxr-xr-x  2 root root  1.0K Jan 10 17:08 opt
lrwxrwxrwx  1 root root     6 Jan 10 17:08 run -> ../run
drwxr-xr-x  3 root root  1.0K Jan 10 17:08 spool
drwxrwxrwt  2 root root  1.0K Jan 10 17:10 tmp
drwxr-xr-x  4 root root  1.0K Jan 21 07:31 usr
```
 
# Adding disks to crypttab and fstab
