# Steam Deck (SteamOS) installing any packages/software
SteamOS has (for the most part) a read-only filesystem which is wiped every update. The reason for this is to ensure stablity and security, espcially since despite being based on Arch Linux, its updates are infrequent and its packages are outdated to retain maximum compatibility.

### So why not just use pacman?

Technically, you can just install packages by simply disabling readonly and installing arch packages the usual way:
* `sudo steamos-readonly disable`
* `sudo pacman -Sy`
* `sudo pacman -S <package name>`

**But don't do this!** What will end up happening is eventually a package you try to install will require a newer version of an existing dependency, and updating it would break compatibility with the Steam Deck causing an unknown amount of problems.

Even if there are no conflicts, your changes will be wiped on the next update - that's annoying!

### A solution?

At this point, most people would suggest just using Flatpaks, AppImages, or Distrobox; what they don't realize is that there already exists an incredibly powerful tool that allows effortless creation and management of Linux containers and we can use it to solve **ALL** of these problems!

# systemd-nspawn!

It's a chroot on steroids that can boot different Linux operating systems. We can run desktop applications from it and even run additional container solutions like Docker and Podman inside. Whatever you could do on a traditional Linux operating system can be done inside an nspawn container. The best part is you do not need to install it.

### Installing a secondary Linux OS to a directory
systemd-nspawn boots directories so we need to install an OS to one. It can be any Linux OS. You can use tools like debootstrap or pacstrap to install from a single command. (just don't use pacstrap from inside SteamOS or it will download from the SteamOS repos)

Btw, I have an Arch Linux install as a btrfs subvolume mounted to /mnt/archlinux.

## Setting up Docker

My life wouldn't be complete without Docker, so as a bonus I'm going to show how to get it working.
