# Steam Deck on SteamOS with LUKS Encryption
This is a Steam Deck SteamOS guide for LUKS encryption which works with the on-screen keyboard and does not require a SteamOS reinstall. 

To make this all possible; Full Disk Encryption simply can't work. We will be partitioning and encrypting the SD card and /home instead, which will still protect your Steam logins, browser cookies, cache and other important user data. 

If you have any other custom partitions which SteamOS does not rely on to run, you can encrypt those as well.

# Prerequisites
* A USB-C hub with USB ports which works in the Steam Deck, preferably with power pass-through support so that you can charge the deck as well (encryption takes a while)
* A USB flash drive with a Linux Installer on it

(While you theoretically could just use an SSH server instead, connect to it from another device, logout of the deck user and unmount the SD card and home partitions; having an installer and hub will ensure that if a fuckup happens, you'll have full ability to restore the deck back to a working state. Locking yourself out of your own deck is not fun.)

* Partition for SD card which will be used for encryption
* Partition for home which will be used for encryption
* Another partition for home which will **not** be encrypted (in this guide, this partition will only be available during decryption, so only give it space for a Steam user install - like 2GB)
* **A complete backup of all the mentioned partitions stored on a different device** since they **will get erased**
* A Throwaway Steam account for the unencrypted home Steam install (this is for best compatibility, trust me)
