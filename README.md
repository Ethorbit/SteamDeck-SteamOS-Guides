# Steam Deck (SteamOS) LUKS Encryption
This is a Steam Deck SteamOS guide for LUKS encryption which works with the on-screen keyboard and does not require a SteamOS reinstall. 

To make this all possible; Full Disk Encryption simply can't work. We will be encrypting the SD card and /home instead, which will still protect your Steam logins, browser cookies, cache and other important user data. 

If you have any other custom partitions which SteamOS does not rely on to run, you can encrypt those as well.

# ![warning-icon](https://i.imgur.com/ZWdfbEN.png) Warning
**None of this is official**. I am not a developer for Valve. The method used here is very hacky, intended only for SteamOS and may not work in future releases. 

I'm not responsible for any damage or data loss.

# Prerequisites
* A **complete backup of all data from your SD card, system /home/ directory and anything else you decide to encrypt, stored on another device.** It's all getting erased.
* A Throwaway Steam account which will only be used to decrypt the disks (this is for best SteamOS compatibility, trust me)

* A USB-C hub with USB ports which work in the Steam Deck, preferably with power pass-through support so that you can charge the deck as well (securely wiping disks takes a long time)
* A USB flash drive with a Linux install(er) on it like [this](https://archlinux.org/download/), so you can access the SteamOS partitions from outside

(While you technically could just do everything inside SteamOS; having an installer and hub will ensure that if a fuckup happens which prevents booting, you'll have full ability to restore the deck back to a working state. It will also ensure that SteamOS will not get in your way during encryption.)

![Example](https://i.imgur.com/W7EdVYn.png)

# Opening Desktop terminal

* Press the Steam button > Power and Switch to Desktop. 
* Find and open Konsole. 

If you haven't already set a password: type `passwd` to do so.


# Enabling TPM and dm_crypt

For some reason, Valve has intentionally disabled TPM and dm_crypt. These are needed for encryption.

Enter these commands:
* `echo "dm_crypt" | sudo tee /etc/modules-load.d/dm_crypt.conf`
* `sudo sed -i "s/module_blacklist=tpm//" /etc/default/grub`
* `sudo sed -i "s/rd.luks=0//" /etc/default/grub`
* `sudo update-grub`

# Adding verbosity to boot

The chances of something going wrong and not being able to boot after this guide is pretty high. It is going to be pretty impossible to determine *why* the system isn't booting if all you see is a logo and then nothing.

Edit 00_header: `sudo nano /etc/grub.d/00_header`

Look for a line called *steamenv_quiet* and place a # at the beginning to comment it out.
Copy paste the steamenv_noisy line and rename the copy from *steamenv_noisy* to *steamenv_quiet*

It should look like this:
```bash
# steamenv_quiet="loglevel=3 splash quiet plymouth.ignore-serial-consoles fbcon=vc:4-6"
steamenv_quiet="loglevel=4 splash=verbose fbcon=nodefer"
steamenv_noisy="loglevel=4 splash=verbose fbcon=nodefer"
```

Update grub: `sudo update-grub`

Before proceeding, **reboot to make sure there is text instead of a logo.**

Note: *There will always be an animation that shows when the deck turns on.*

[![Example](https://i.imgur.com/4NxaV5I.png)

# Booting into a Linux installer
Despite its name, we are not actually going to be "installing" anything; we just need a separate temporary Linux environment to use while we manage and encrypt partitions. We are doing it this way to simply avoid any file conflicts. My flash drive has Ventoy with an Arch Linux Installer .iso on it, so I will be booting from that.

With the Deck powered off and usb devices plugged in, hold the Volume Down and Power button at the same time until you hear a noise. Select the Flash drive to boot into it.

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
  
We only care about the **home** partition of the nvme drive.

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
#### Unencrypted home
* Enter n, press enter twice until it asks for "Last Sector"
* For Last Sector, I will enter +10G. +10G means I'm giving 10 gibibytes of space for the unencrypted home partition, which will only need enough space for a Steam install.
	
At the time of writing: A Steam install costs about ~7.7G of space, and the swapfile SteamOS creates costs 1G.
I've chosen 10G to give a little head room, but I will also be applying aggressive compression with btrfs in later steps to help save even more space.

#### Encrypted home
* Enter n and press enter until it stops asking. This will be the encrypted home where you'll *actually* store files.
* Enter p to check if your partitions look good. 
```
Device            Start       End  Sectors  Size Type
/dev/nvme0n1p1     2048    133119   131072   64M EFI System
/dev/nvme0n1p2   133120    198655    65536   32M Microsoft basic data
/dev/nvme0n1p3   198656    264191    65536   32M Microsoft basic data
/dev/nvme0n1p4   264192  10749951 10485760    5G Linux root (x86-64)
/dev/nvme0n1p5 10749952  21235711 10485760    5G Linux root (x86-64)
/dev/nvme0n1p6 21235712  21759999   524288  256M Linux variable data
/dev/nvme0n1p7 21760000  22284287   524288  256M Linux variable data
/dev/nvme0n1p8 22284288  43255807 20971520   10G Linux filesystem
/dev/nvme0n1p9 43255808 120829951 77574144   37G Linux filesystem
```
  
* Enter w to write changes
  
# Encrypting
My /dev/mmcblk0p1 and /dev/nvme0n1p8 are the two partitions that need to be encrypted.
  
![warning-icon](https://i.imgur.com/ZWdfbEN.png) **Make sure you double check that the partitions you are about to encrypt are the ones YOU created**, these next actions have the potential to bork your whole SteamOS install (if passed the wrong partition(s).)

We will use cryptsetup luksFormat to setup encryption for them. 
* `cryptsetup luksFormat /dev/mmcblk0p1`
* `cryptsetup luksFormat /dev/nvme0n1p9`
  
It will ask you to confirm YES and then to enter a secure, memorable password. If you forget this password, you're screwed. Keep backups of important files.
  
This is optional, but you can also give them labels:
* `cryptsetup config /dev/mmcblk0p1 --label crypt_sdcard`
* `cryptsetup config /dev/nvme0n1p9 --label crypt_home`
  
Next we will open them
* `cryptsetup luksOpen /dev/mmcblk0p1 crypt_sdcard`
* `cryptsetup luksOpen /dev/nvme0n1p9 crypt_home`

We should never use them directly as they are now encrypted. The LUKS mappings (opened) devices are located at /dev/mapper/crypt_sdcard and /dev/mapper/crypt_home.
  
The next thing we will do is securely wipe them. **This is a process that will take a long time to complete**, especially with the slower SD card - which is why you should be charging your deck by now.

* `dd if=/dev/zero of=/dev/mapper/crypt_sdcard bs=1M status=progress`
* `dd if=/dev/zero of=/dev/mapper/crypt_home bs=1M status=progress`
  
Normally, this operation would allocate the entire disk with zeroes, but since all writes to these devices are encrypted, these zeroes will also be encrypted. This makes it way harder for attackers to figure out which parts of the encrypted disks actually contain user data.
  
# Making filesystems

Use `mkfs` to make a filesystem for the partitions you created, so that you can actually mount and write to them later.

I will mostly be using the Btrfs filesystem since it supports transparent compression, subvolumes, deduplication, snapshotting, and more. The compression specifically is useful, because it will help save space with no effort.

Note: *SteamOS's /home by default uses the ext4 filesystem (which is stable and fast), but missing the optional features mentioned.*
  
The unencrypted home:

* `mkfs.btrfs /dev/nvme0n1p8`
* `btrfs filesystem label /dev/nvme0n1p8 unencrypted_home`

The LUKS mappings:

* `mkfs.btrfs /dev/mapper/crypt_sdcard`
* `mkfs.btrfs /dev/mapper/crypt_home`
  
# Mounting SteamOS /var partition

So SteamOS's /var directory is **very** different than what you'd normally expect from a typical distro. Most of SteamOS runs on an immutable filesystem which is overriden each update, so the /var directory was repurposed to just be "the place" for the user to modify some parts of the system at. Because of this, we will be making pretty much all file changes on /var.
 
```
root@archiso ~ # lsblk -o name,label,size                       
NAME             LABEL              SIZE
loop0                             689.8M
sda                                   0B
sdb                                57.8G
├─sdb1           Ventoy            57.7G
│ └─ventoy       ARCH_202207      795.3M
└─sdb2           VTOYEFI             32M
mmcblk0                           477.5G
└─mmcblk0p1      crypt_sdcard     477.5G
  └─crypt_sdcard                  477.5G
nvme0n1                       57.6G
├─nvme0n1p1 esp                 64M
├─nvme0n1p2 efi                 32M
├─nvme0n1p3 efi                 32M
├─nvme0n1p4 rootfs               5G
├─nvme0n1p5 rootfs               5G
├─nvme0n1p6 var                256M
├─nvme0n1p7 var                256M
├─nvme0n1p8 unencrypted_home    10G
└─nvme0n1p9 crypt_home          37G
  └─crypt_home                     37G
```  
  
As you can see, there are are two SteamOS partitions with the label 'var' and size of 256M: mount the first one.
  
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
 
# Making disks automatically mount

The cryptsetup service checks for disks in crypttab, we can invoke this service later to get a handy passphrase prompt for all disks to decrypt. The cryptsetup service caches passphrases, so if multiple disks have the same one, you only need to enter it once. 
  
If we add the LUKS mapping paths to fstab, we can automatically mount them after decrypting the partitions.
  
Instead of referencing the device names, we will be using their UUIDs. You can view the UUID of every disk by entering: `lsblk -o name,label,size,fstype,type,uuid`
```
root@archiso ~ # lsblk -o name,label,size,fstype,type,uuid
NAME             LABEL              SIZE FSTYPE      TYPE  UUID
loop0                             689.8M squashfs    loop  
sda                                   0B             disk  
sdb                                57.8G             disk  
├─sdb1           Ventoy            57.7G exfat       part  8623-8A3A
│ └─ventoy       ARCH_202207      795.3M iso9660     dm    2022-07-01-13-20-00-00
└─sdb2           VTOYEFI             32M vfat        part  5A89-BA75
mmcblk0                           477.5G             disk  
└─mmcblk0p1      crypt_sdcard     477.5G crypto_LUKS part  c29abde6-8237-410e-a338-f808ff065c99
  └─crypt_sdcard                  477.5G btrfs       crypt 80c91f87-2164-4286-8c2e-6d317849b262
nvme0n1                       57.6G             disk 
├─nvme0n1p1 esp                 64M vfat        part 89B1-076D
├─nvme0n1p2 efi                 32M vfat        part 89B1-BC90
├─nvme0n1p3 efi                 32M vfat        part 89B2-685B
├─nvme0n1p4 rootfs               5G btrfs       part 1fae9051-7984-45be-9600-865a94ad8808
├─nvme0n1p5 rootfs               5G btrfs       part 4a39a236-7977-4eb7-8ea1-2a9c41ff7fee
├─nvme0n1p6 var                256M ext4        part c6e05ac9-5103-4808-9b5f-0d3de52a16e6
├─nvme0n1p7 var                256M ext4        part 1041ad1e-13e1-4f86-ac15-fee153092189
├─nvme0n1p8 unencrypted_home    10G btrfs       part 39e8f2ac-b3f4-427d-8f3b-b3165612d232
└─nvme0n1p9 crypt_home          37G crypto_LUKS part d5eaf671-f52d-4cf4-b912-2a66834ff1dc
│ └─crypt_home                       45G btrfs       crypt 8694c03f-64e9-4601-bb97-f6cd4a3b3d5a
```
 
As you can see, the partitions we encrypted appear with the type "crypto_LUKS". Your UUIDs will be different than mine, **do not use mine.**
  
### Crypttab
Edit crypttab: `nano /mnt/lib/overlays/etc/upper/crypttab` (**use the partition UUIDs**, not the LUKS mapping ones)
```
crypt_home      UUID="b27f07a2-f2be-49b1-b769-c67d9ab2eb98"     none    luks,discard,noauto,nofail,noearly
crypt_sdcard    UUID="c29abde6-8237-410e-a338-f808ff065c99"     none    luks,noauto,nofail,noearly
```
 
#### `discard`
The `discard` is there to enable TRIM support for the NVMe. This will make the dd operation we did meaningless with time, which will make it easier for attackers to know what parts of the disk has been free'd and such. You can remove it if you want, but just know that not having TRIM enabled for an SSD can alter performance while off and may decrease longevity of the drive. Your data will still be encrypted regardless.
  
After data has been written, you cannot undo exposure by simply disabling it, unless the device is erased again.

### Fstab
Edit fstab: `nano /mnt/lib/overlays/etc/upper/fstab`
  
Replace the existing /home line with this:
```
# Unencrypted home
UUID="f80b8cb3-0dbf-4cd9-8db4-29c78cfa3266"     /home     btrfs   defaults,force-compress=zstd:16       0       2
```
	
Note: **force-compress is a btrfs option**, remove it if you're not using btrfs.

### Mount script: `nano /mnt/usr/sbin/mount-encrypted.sh`
```bash
#!/bin/bash  
mount /dev/mapper/crypt_home /home
mount /dev/mapper/crypt_sdcard /var/mnt/sdcard
```
    
`chmod 755 /mnt/usr/sbin/mount-encrypted.sh`
	
This mount script's crypt_home entry should override the unencrypted home, so that when it's mounted, programs will write to the encrypted home rather than the unencrypted one.

# Creating decrypt script

SteamOS treats user home directories kinda like system installs. When a user has an empty home directory, it displays an install prompt and downloads steam files with an eventual Login panel appearing.

The decrypt script will run the cryptsetup service. After a successful unlock, your mount script you created earlier will run and all programs by every user except root will be terminated. After the programs are relaunched, they will write to the encrypted directories instead, keeping all your data safe.
  
### Creation
* `nano /mnt/usr/sbin/decrypt.sh`
```bash
# to be entered
```
* `chmod 755 /mnt/usr/sbin/decrypt.sh`

This is a root-only script. You won't be decrypting the system as root, so we will add a sudoer entry for the default user:

`echo "deck ALL=(root) NOPASSWD: /var/usr/sbin/decrypt.sh" >> /mnt/lib/overlays/etc/upper/sudoers`

**If your username was changed from the default 'deck', make sure to change it there too.**
  
We will want a decryption prompt as soon as we open terminal, so let's mount the unencrypted home directory and add the .bashrc:
* `umount /mnt`
* `mount /dev/disk/by-label/unencrypted_home /mnt`
* `mkdir /mnt/deck`
* `chown 1000:1000 /mnt/deck`
* `chmod 700 /mnt/deck`
* `echo "sudo /var/usr/sbin/decrypt.sh" > /mnt/deck/.bashrc`
* `chown 1000:1000 /mnt/deck/.bashrc`
  
Again, make sure to use *your* device ID and replace 'deck' if your username is different. You can check if 1000 is the right UID with: `cat /mnt/lib/overlays/etc/upper/passwd | cut -d ":" -f 1,3`
  
It is time to boot back into SteamOS: `shutdown -r now`
  
# Booting back into the system
  
If everything worked, the screen will be black for a few minutes and then a Steam setup menu should appear.
  
![Example](https://i.imgur.com/LpVYZiX.png)
	
### Create a disposable Steam account
This Steam install simply exists to decrypt the system every time you boot, you will **not** use it to play games (the unencrypted partition only has enough space for the Steam install anyway)
  
Login using a throwaway Steam account (if you do not have one, create a new one). Since this partition is not encrypted, you don't want to login with anything you care about.

### Attempt decryption
With your new account, go to the Desktop and open Konsole. There should be a LUKS password prompt. As soon as you enter the password, your screen should go black for a few minutes like before, and another Steam Setup menu will appear. This time, your disposable account should not exist, which means that you are now using the **encrypted** disks, and you can feel free to do whatever now.
  
#### Adding Konsole as a non-steam game
If you don't like having to switch to Desktop every time to decrypt, you can bring Konsole into your Steam library too. 
  
* Go to the Desktop and open Steam. 
* On the bottom left, click [+] Add A Game
* Click Add a Non-Steam Game
* Find and select Konsole from the list
* Right-click Konsole from the library
* Click Properties
* Under Launch Options, add: `--fullscreen`
* Test if it works
	
# Troubleshooting

### Unable to boot, nothing works, black screen
If you experience lots of boot errors that pass really fast and then the screen stays black - Something's wrong with the unencrypted home partition. SteamOS actually creates and uses directories inside /home/.steamos/ as bind mountpoints for system directories, so the system simply won't work correctly without it.
	
If you didn't experience boot errors, this can instead mean that there is not enough space on the disk for the Steam setup. If that's the case, you'll need to sadly re-configure everything related to disks to give the unencrypted home more space.
	
### Unable to login to Steam
Do Power > Switch to Desktop, plug a mouse in and then login to Steam there. The reason I mention a mouse is because for some reason, the native Steam Deck controls do not fully work in Desktop mode until **after** a Steam account has been added.
