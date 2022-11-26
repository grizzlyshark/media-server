# Media Server Install Guide

## Install ArchLinux

Get the [latest image](http://mirrors.mit.edu/archlinux/iso/latest/archlinux-x86_64.iso) and build a bootable USB. Boot from USB and at prompt, type `archinstall`.

Configure the following:

**Drives:**

- Select the drive you want to boot from

**Disk layout:**

- Select `Wipe all selected drives...`
- Select `ext4`
- Do not create /home partition

**Hostname:**

- Set this to what you want your server to be named

**User Account:**

- Add a User
- Choose yes when prompted to be a superuser

**Network configuration:**

- Select `Manual configuration`
- Choose `Add interface`
- Select your wired interface. Probably `enp3s0`
- Select `DHCP`
- Select `Confirm and exit`

**Select `Install`:**

- Your configuration will be displayed in JSON. Press `Enter` to continue

Once installation is completed, reboot and login.

---

## Setup port forwarding

On your router, assign a static IP to your server and forward ports 8080, 8081, and 8082 (or the 3 ports you chose) to your server's IP address.

---

## Install a text editor

**Note:** If you don't like vim, replace vim with nano in this guide. If you opened vim and can't figure out how to close it, enter `:q!` and press `Enter`.

```bash
sudo pacman -S nano vim
```

---

## Install firewall and OpenSSH

We are going to remove iptables, and replace it with nftables.

**Install nftables:**

```bash
sudo pacman -S iptables-nft
```

Enter `y` when prompted to remove iptables.

**Enable and start nftables:**

```bash
sudo systemctl enable nftables
sudo systemctl start nftables
```

**Open nftables to media server ports:**

Add the following line to `/etc/nftables.conf`:

> `tcp dport {8080, 8081, 8082, 32400} accept comment "media server"`

```bash
sudo vim /etc/nftables.conf
```

Your `/etc/nftables.conf` should look like:

```nft
#!/usr/bin/nft -f
# vim:set ts=2 sw=2 et:

# IPv4/IPv6 Simple & Safe firewall ruleset.
# More examples in /usr/share/nftables/ and /usr/share/doc/nftables/examples/.

table inet filter
delete table inet filter
table inet filter {
  chain input {
    type filter hook input priority filter
    policy drop

    ct state invalid drop comment "early drop of invalid connections"
    ct state {established, related} accept comment "allow tracked connections"
    iifname lo accept comment "allow from loopback"
    ip protocol icmp accept comment "allow icmp"
    meta l4proto ipv6-icmp accept comment "allow icmp v6"
    tcp dport ssh accept comment "allow sshd"
    tcp dport {8080, 8081, 8082, 32400} accept comment "media server"
    pkttype host limit rate 5/second counter reject with icmpx type admin-prohibited
    counter
  }
  chain forward {
    type filter hook forward priority filter
    policy drop
  }
}
```

**Install OpenSSH:**

```bash
sudo pacman -S openssh
sudo systemctl enable sshd
sudo systemctl start sshd
```

​You should be able to SSH into the server now. You can do so by entering `ssh username@hostname` into your terminal, where **username** is the username you created and **hostname** is your server name or IP address.

---

## Create users and groups

You will need to create users and groups for the services, and make sure everyone is part of the media group.

**Create users:**

```bash
sudo useradd -r -s /usr/bin/nologin sabnzbd
sudo useradd -r -s /usr/bin/nologin radarr
sudo useradd -r -s /usr/bin/nologin sonarr
sudo useradd -r -s /usr/bin/nologin plex
sudo groupadd media
```

**Add users to groups:**

Replace $user with your username

```bash
sudo gpasswd -a $user media
sudo gpasswd -a sabnzbd media
sudo gpasswd -a radarr media
sudo gpasswd -a sonarr media
sudo gpasswd -a plex media
```

You must log out and back in for this change to take effect.

---

## Mount storage

I am not using raid because I want to maximize storage and am not hosting anything critical. You could also use a volume group to manage your drives as one storage structure, or configure raid for redundancy.

Your drives should be already formatted to ext4 preferrably. This guide can help you format your drives if needed:
[Use fdisk Format Partition](https://linuxhint.com/use-fdisk-format-partition/)

First identify your drives with `lsblk`:

```bash
[dave@snickers ~]$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   7.3T  0 disk
└─sda1   8:1    0   7.3T  0 part
sdb      8:16   0   7.3T  0 disk
└─sdb1   8:17   0   7.3T  0 part
sdc      8:32   0   1.8T  0 disk
└─sdc1   8:33   0   1.8T  0 part
sdd      8:48   0 447.1G  0 disk
├─sdd1   8:49   0   200M  0 part /boot
└─sdd2   8:50   0 446.9G  0 part /
zram0  254:0    0   3.4G  0 disk [SWAP]
```

Ignore any device with recorded `MOUNTPOINTS`. What we are looking for are the partitions that have not been mounted. In the example above, they are:

- sda1
- sdb1
- sdc1

sdd can be ignored as it is the boot drive as seen by having a partition with a /boot and / mountpoints.

### Create mount points for drives

For each drive that you are mounting, you will need to create a directory in which to mount it.

I am mounting my drives in /media, into directories named media{1-3}.

Create the directories with `mkdir`:

```bash
sudo mkdir /media /media/media1 /media/media2 /media/media3
```

### Mount the drives

Mount each of your drives to its corresponding directory.

**Note:** Do not copy and paste these commands. Use the devices (/dev/sdx) that your system reports.

```bash
sudo mount /dev/sda1 /media/media1
sudo mount /dev/sdb1 /media/media2
sudo mount /dev/sdc1 /media/media3
```

Take ownership of the mounted drives by entering the following, where \$username is your username. Use the directories that you mounted the drives in, separated by a space:

```bash
sudo chown -R $username:media /media/media1 /media/media2 /media/media3
sudo chmod -R a+rx /media/media1 /media/media2 /media/media3
sudo chmod -R g+wrx /media/media1 /media/media2 /media/media3
```

**Add mounts to fstab:**

This will ensure that your drive mounts properly on system reboot.

View your mounted drives with `cat /etc/mtab`. At the bottom of the output you should see the drives you just mounted:

```fstab
[dave@snickers ~]$ cat /etc/mtab
...
...
/dev/sda1 /media/media1 ext4 rw,relatime 0 0
/dev/sdb1 /media/media2 ext4 rw,relatime 0 0
/dev/sdc1 /media/media3 ext4 rw,relatime 0 0
```

Copy the lines that correspond to your mounted drives and paste them into the bottom of `/etc/fstab`:

```bash
sudo vim /etc/fstab
```

---

## Install and configure SABnzbd

Full instructions [here](https://wiki.archlinux.org/title/SABnzbd).

**Install dependencies:**

```bash
sudo pacman -S python-pyopenssl p7zip unzip git base-devel
```

**Install yay to manage AURs:**

```bash
git clone https://aur.archlinux.org/yay.git
cd yay/
makepkg -sri
yay
cd
rm -rf ~/yay
```

**Install SABnzbd:**

```bash
sudo yay -S sabnzbd
sudo systemctl enable sabnzbd
sudo systemctl start sabnzbd
```

**Edit SABnzbd config:**

Edit the `host` entry to read `host = $server_ip` in `sabnzbd.ini`, where `$server_ip` is your server's IP address

```bash
sudo vim /var/lib/sabnzbd/sabnzbd.ini
```

Restart SABnzbd

```bash
sudo systemctl restart sabnzbd
```

**Edit primary user group:**

```bash
sudo usermod -g media sabnzbd
sudo groupdel sabnzbd
```

### Configure SABnzbd

You should now be able to access the web GUI at `http://$server_ip:8080`.

- Select your language and click `Start Wizard`
- Enter your Usenet provider's information and make sure `Test Server` returns `Connection Successful!`
- Configure your `Completed Download` and `Temporary Download` folders if you don't want them on your boot drive.
  - Radarr/Sonarr will move them to your library automatically after completion.
- Select `Go to SABnzbd`
- Click on the gear icon in the upper right and configure to your preferences
  - I recommend setting a username and password
  - **Note:** Click through the tabs at the top so that your configs are generated. If you don't visit the `categories` tab first, you may need to come back if Radarr can't detect SABnzbd's categories
- Take note of your API key in the `General` tab or enter the following in your terminal:
  - > `cat ~/.sabnzbd/sabnzbd.ini | grep api_key`
- Click on the `Folders` tab and enter `775` in the `Permissions for completed downloads`
  - Scroll to the bottom of this section and click `Save Changes`

---

## Install and configure Radarr

Full instructions [here](https://radarr.video/#downloads-v3-linux).

**Install Radarr:**

```bash
yay -S radarr
```

**Start Radarr:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now radarr
```

**Edit Radarr config:**

Change the `Port` tag from `7878` to `8081` in `config.xml`

```bash
sudo vim /var/lib/radarr/config.xml
```

**Restart Radarr:**

```bash
sudo systemctl restart radarr
```

### Configure Radarr

You should now be able to access the web GUI at `http://$server_ip:8081`.

**Add Libraries:**

- Click `Settings` on the navigation bar, then click `Media Management`
- Click `Add Root Folder`
- Select the root folder that your movies are in. This is also where Radarr will place movies
  - You can do this for each drive

**Configure Indexers:**

- Select `Indexers` in the navigation bar
- Click the `+` icon
- For NZBgeek, click the `presets` dropdown for `Newznab`
  - Select `NZBgeek` from the dropdown list
- Retrieve your `geek (api) key` from [NZBgeek](https://nzbgeek.info/dashboard.php?myaccount) and enter it into Radarr's `API Key` field
- Ensure that the `Test` button returns a `✓` and click `Save`

**Configure Download Client:**

- Select `Download Clients` from the navigation bar
- Click the `+` icon
- Click `SABnzbd`
- Enter `SABnzbd` in the name field
- Enter your server's IP address in the `Host` field
- Retrieve your API key from SABnzbd and enter it in the `API Key` field
  - To easily retrieve your key:
    > `sudo cat /var/lib/sabnzbd/sabnzbd.ini | grep api_key`
- Your SABnzbd username and password are not necessary

**Configure Permissions:**

- Select `Media Management` from the navigation bar
- Click the `Show Advanced` gear icon in the upper right to view advanced options
- In the `Permissions` section, check the `Set Permissions` box
- In the `chmod Folder` field, enter `775`
- In the `chown Group` field, enter `media`
- Click the `Save Changes` button at the top of the page

---

## Install and configure Sonarr

Full instructions [here](https://sonarr.tv/#downloads-v3-linux-archlinux).

**Install Sonarr:**

```bash
yay -S sonarr
```

**Start Sonarr:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sonarr
```

**Edit Sonarr config:**

Change the `Port` tag from `8989` to `8082` in `config.xml`

```bash
sudo vim /var/lib/sonarr/config.xml
```

**Restart Sonarr:**

```bash
sudo systemctl restart sonarr
```

### Configure Sonarr

You should now be able to access the web GUI at `http://$server_ip:8082`.

**Add Libraries:**

- Click `Settings` on the navigation bar, then click `Media Management`
- Click `Add Root Folder`
- Select the root folder that your shows are in. This is also where Sonarr will place shows
  - You can do this for each drive

**Configure Indexers:**

- Select `Indexers` in the navigation bar
- Click the `+` icon
- For NZBgeek, click the `presets` dropdown for `Newznab`
  - Select `NZBgeek` from the dropdown list
- Retrieve your `geek (api) key` from [NZBgeek](https://nzbgeek.info/dashboard.php?myaccount) and enter it into Sonarr's `API Key` field
- Ensure that the `Test` button returns a `✓` and click `Save`

**Configure Download Client:**

- Select `Download Clients` from the navigation bar
- Click the `+` icon
- Click `SABnzbd`
- Enter `SABnzbd` in the name field
- Enter your server's IP address in the `Host` field
- Retrieve your API key from SABnzbd and enter it in the `API Key` field
  - To easily retrieve your key:
    > `sudo cat /var/lib/sabnzbd/sabnzbd.ini | grep api_key`
- Your SABnzbd username and password are not necessary

**Configure Permissions:**

- Select `Media Management` from the navigation bar
- Click the `Show Advanced` gear icon in the upper right to view advanced options
- In the `Permissions` section, check the `Set Permissions` box
- In the `chmod Folder` field, enter `775`
- In the `chown Group` field, enter `media`
- Click the `Save Changes` button at the top of the page

---

## Install and configure Plex Media Server

**Install Plex Media Server:**

```bash
yay -S plex-media-server
```

**Start Plex:**

```bash
sudo systemctl enable --now plexmediaserver
```

**Configure Plex:**

- Visit `http://$server_ip:32400/web` and run through the setup wizard.
- Make sure to add all of the `movies` and `tv` folders that you created.
  - **Note:** Do not add the root `/media` or `/media#` folders if you have SABnzbd configured to manage downloads in `/media`
- Finish configurations to your preference

---

## Final configurations and suggestions

You should now have a basic, working SABnzbd/Radarr/Sonarr/Plex server.

Yar!

### SABnzbd folders

I recommend changing the default folders for temporary and completed downloads so that they are not on your boot drive. By default, SABnzbd will place these folders in `/var/lib/sabnzbd`. Instead, I chose to put them on one of my storage drives to avoid clutter and to segregate media from operations.

### Storage architecture

You will need to put some consideration into your storage architecture before proceeding. My setup is described below with some thoughts on my choices.

- I have 3 storage drives that I am using in a non-raid setup
- All 3 drives are mounted to `/media`
- Each drive has a `movies` and `tv` directory for Sonarr/Radarr to utilize
- One of my drives was selected to manage SABnzbd downloads and contains a directory named `sab`, with subdirectories `downloading` and `completed` for SABnzbd to utilize
  - The `sab` subdirectories `downloading` and `completed` may eventually need to be manually purged if your system abandons a download due to a crash, a bad download, or just bad luck
  - SABnzbd, Radarr, and Sonarr are typically pretty good about cleaning up things but sometimes things are missed or the service is unable to complete its task for whatever reason
- This solution has zero redundancy. If a drive dies, I will lose everything on it. This is acceptable to me as the drives only contain media and it allows me to maximize storage space and not have to deal with raid overhead

Here is a representation of my storage solution:

```tree
/media
├── media1
│   ├── movies
│   ├── sab
│   │   ├── completed
│   │   └── downloading
│   └── tv
├── media2
│   ├── movies
│   └── tv
└── media3
    ├── movies
    └── tv
```

### Tweaking

You will have to play around with Radarr and Sonarr's `Quality` and `Profile` settings to find what works right for you. Depending on your storage space, you may not want to download 1080p movies that are 30GB causing your space to fill up quickly. Similarly with TV shows - if your settings allow Sonarr to download episodes that are 10GB each, you may run out of space faster than expected.

You will need to experiment to find a happy medium. You can adjust both `Quality` and `Profile` settings to dial things in to your specific needs. You can also diable or create new profiles to suit you better than the defaults. I typically have 2 profiles active for TV and movies - 1 configured for 1080p HD at a reasonable quality, and another for 720p and below that I can use for older media or things that I don't care about having in high quality.

I recommend setting a maximum line speed in SABnzbd. This will allow you to throttle your connection. By default, it uses full bandwidth and may noticeably slow down other traffic on your network. You can also set schedules so that bandwidth will be limited during set hours, and opened during other hours. You can also set total monthly bandwidth limits if you are concerned with hitting your ISP's data cap.

---

## Documentation

[ArchLinux Wiki](https://wiki.archlinux.org/)

[SABnzbd Wiki](https://sabnzbd.org/wiki/)

[Radarr Wiki](https://wiki.servarr.com/en/radarr)

[Sonarr Wiki](https://wiki.servarr.com/en/sonarr)
