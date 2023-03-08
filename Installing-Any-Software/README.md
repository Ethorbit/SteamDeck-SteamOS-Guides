This is unfinished.

# Steam Deck (SteamOS) installing any packages/software
SteamOS has (for the most part) a read-only filesystem which is wiped every update. The reason for this is to ensure stablity and security, especially since despite being based on Arch Linux, its updates are infrequent and its packages are outdated.

### So why not just use pacman?

Technically, you can just install packages by simply disabling readonly and installing arch packages the usual way:
* `sudo steamos-readonly disable`
* `sudo pacman-key --init`
* `sudo pacman-key --populate archlinux`
* `sudo pacman -S <package name>`

**But don't do this!** What will end up happening is eventually a package you try to install will require a newer version of an existing dependency, and updating it would break compatibility with the Steam Deck causing an unknown amount of problems.

Even if there are no conflicts, your changes will be wiped on the next update - that's annoying!

### A solution?

At this point, most people would suggest just using Flatpaks, AppImages, or Distrobox; what they don't realize is that there already exists an incredibly powerful tool that allows effortless creation and management of Linux containers and we can use it to solve **ALL** of these problems!

# systemd-nspawn!

It's a chroot on steroids that can boot different Linux operating systems. We can run desktop applications from it and even run additional container solutions like Docker and Podman inside. Whatever you could do on a traditional Linux operating system can be done inside an nspawn container. The best part is you do not need to install it.

Before proceeding, set a password if you haven't already. Inside the Desktop Konsole, type: `passwd` to set it.

## ![warning-icon](https://i.imgur.com/ZWdfbEN.png) Warning
Because the goal is to make the nspawn container tightly integrated with the host and to give it the greatest privileges so that it can run anything, you have the ability to destroy your SteamOS system from inside. You should still take care what root commands you execute inside the container and treat it as if you were using your host system. You don't have to worry about file conflicts though because we will only share /home and mounts, so feel free to install anything. Just understand that if you wanted to, you could completely wipe your home with a single root command just like you could if you were on your host.

I'm not responsible for any damage or data loss.


## Installing a secondary Linux OS to a directory
systemd-nspawn boots directories so we need to install an OS to one. It can be any Linux OS. You can use tools like debootstrap for Debian, pacstrap for Arch, etc to install from a single command. (just don't use pacstrap from inside SteamOS or it will download from SteamOS repos)

Btw, I have an Arch Linux install as a btrfs subvolume mounted to /mnt/archlinux.

## Creating .nspawn file
We are going to create a systemd .nspawn configuration.

* `sudo mkdir /etc/systemd/nspawn`
* `sudo nano /etc/systemd/nspawn/archlinux.nspawn`
```
[Exec]
Boot=true
Capability=all

[Files]
Bind=/home
Bind=/mnt
Bind=/etc/hosts
Bind=/etc/passwd
Bind=/etc/shadow
Bind=/etc/group
Bind=/etc/gshadow
Bind=/etc/subgid
Bind=/etc/subuid
Bind=/etc/sudoers
Bind=/etc/sudoers.d
BindReadOnly=/etc/resolv.conf
BindReadOnly=/tmp/.X11-unix
```

If you don't want the container to have the ability to manage the deck's users and groups, you should replace Bind with BindReadOnly for hosts, passwd, shadow, gshadow, group, subuid, subgid, sudoers and sudoers.d.

If you don't need the fullest of root privileges for anything, you can remove the Capability=all

We also give it /home and /mnt to share our files with the container.

## Creating history files
So since we plan to share our home directory with our container, we need to make a history file for it, otherwise your command history will be shared with the container. You might be okay with that, but it might become confusing if you forget what command came from where.

* `mv ~/.bash_history ~/.bash_history_deck`
* `touch ~/.bash_history_nspawn`

## Editing .bashrc
This file is executed with every new shell, and since we're sharing our home with the container, the container will also run it.
`nano ~/.bashrc`

```
# If not running interactively, don't do anything
[[ $- != *i* ]] && return
```

First, we need to add a DISPLAY variable. The reason is so that both the host and container share the same display, this is what allows us to open desktop applications combined with our .X11-unix bind we made in our .nspawn config

```bash
export DISPLAY=:0
```

Now here's the problem, how can we tell when we are working with the deck or an nspawn container? Well, one way is we can check the /etc/hostname file. Your deck's hostname by default is "steamdeck". You can check by running this command: `cat /etc/hostname`

```bash
if [[ -f /etc/hostname ]] && 
   [[ `cat "/etc/hostname"` = "steamdeck" ]]
then
    # code for steamOS
    
else
    # code for nspawn container
fi
```

In the steamOS section, we need to add a few lines
* Set our new HISTFILE: `export HISTFILE="$HOME/.bash_history_deck"`
* Container alias: `alias archlinux="sudo systemd-nspawn --machine archlinux -D /mnt/archlinux"`
Make sure to set -D to where you installed your secondary OS. --machine name should be the name of your .nspawn file.
* Allow our user in the container to access our running X display server: `xhost +si:localuser:$USER`

In the nspawn section, we need to set its HISTFILE: `export HISTFILE="$HOME/.bash_history_nspawn"`

Your .bashrc should look like this in the end:
```bash
#
# ~/.bashrc
#

# If not running interactively, don't do anything
[[ $- != *i* ]] && return

export DISPLAY=:0

if [[ -f /etc/hostname ]] && 
   [[ `cat "/etc/hostname"` = "steamdeck" ]]
then
    export HISTFILE="$HOME/.bash_history_deck"
    alias archlinux="sudo systemd-nspawn --machine archlinux -D /mnt/archlinux"
    xhost +si:localuser:$USER
else
    export HISTFILE="$HOME/.bash_history_nspawn"
fi
```

You'll of course need to improve this solution if you plan to run multiple nspawn containers, but as you'll see later you only really need one.

## Booting the container
Just type your command's alias set in .bashrc: `archlinux`

You should see the systemd boot sequence in the terminal as if we were booting a real Linux OS. Now you can update it and install packages as if it were your real OS.

### Testing desktop applications

Install and run xeyes:
* `sudo pacman -S xorg-xeyes`
* `xeyes`

You should see the graphical window pop up on your SteamOS desktop as if you started it from SteamOS. Pretty cool, huh?

### Setting up Docker (Optional)

Yo dawg, I heard you like containers. 

Inside of your .nspawn file:
* under [Exec] add: `SystemCallFilter=add_key keyctl bpf`
* under [Files] add: `Bind=/dev/fuse`

If you made the user directory mounts read-only, you'll need to first add the docker group on the host or the service will fail:
* `sudo groupadd -r docker`
* `sudo usermod -aG docker deck`
* `sudo systemctl restart sddm`

Inside the container, install docker:
* `sudo pacman -S docker`
* `sudo systemctl enable docker --now`

Restart container and you should have a functioning docker inside, if your user is a part of the docker group, you should be able to use docker without root. If it doesn't work, try checking `sudo journalctl -xeu docker` and `sudo dmesg` as it's likely due to a missing kernel parameter.

`docker run -it --rm --name alpine alpine:latest /bin/sh`

Now I'm inside an Alpine Linux Docker container inside of an Arch Linux nspawn container that's inside of SteamOS. The possibilites are endless!

## Conclusion
By default, the Deck can already run any Linux OS and application, you just have to set it up. That makes this quite possibly the most useful handheld device.

nspawn is partly built for security, so as you saw when setting up Docker, we had to still grant additional device access in order to get Docker running properly even though nspawn already runs as root and we granted Capability=all. This might be an issue you may run into when setting up other similarly sophisticated Linux services, but just keep an eye on the logs to see what the applications require that the container may be missing. With the right configuration, nspawn can run anything. 


