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

(While you theoretically could just do everything inside SteamOS; having an installer and hub will ensure that if a fuckup happens, you'll have full ability to restore the deck back to a working state.)

![Example](https://i.imgur.com/W7EdVYn.png)

## Booting into external system
With the Deck powered off and usb devices plugged in, hold the Volume Down and Power button at the same time to reach the boot menu. Select the Flash drive

[![Video Tutorial](https://i.imgur.com/6VH6ZiY.jpg)](https://www.youtube.com/watch?v=2_Pkv4tr8Ho)

From here on out, everything will be done in the terminal.

# Partitioning
To identify storage devices

`lsblk`

* Partition for SD card which will be used for encryption
* Partition for home which will be used for encryption
* Another partition for home which will **not** be encrypted (this partition will only be used for decryption, so only give it space for a Steam user install - like 2GB)

## Encrypting
