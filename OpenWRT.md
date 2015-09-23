# OpenWRT SFTP & BitTorrent server
This guide is intended to walk a user through building a router-based multi-user SFTP & BitTorrent (using [Transmission](http://www.transmissionbt.com/)) server with [OpenWRT](https://openwrt.org/). You should have a working router running OpenWRT (and LuCI) before starting. Upon completion, you will have a router with seedbox capabilities. SFTP users (that you manage) will be able to download files from shared directories without having router shell access.

Don't need BitTorrent transfers? Skip The Transmission section and transfer local data to the SFTP server via Samba.

> #### NOTE!
> * This guide assumes you are familiar with command-line terminals and VIM or another command-line text editor
> * Text within brackets such as **_[username]_** are variables to be replaced at your discretion
> * All commands are to be run as the root user (use ``` sudo -i ``` after disabling root SSH access)

## Prerequisites
### Hardware
* USB storage drive formatted EXT4 **(NTFS-3G is a stable read/write NTFS driver, but is too slow for this implementation)**
* USB swap drive formatted SWAP **(optional)**

### Software
Via SSH, update OPKG (Open PacKaGe Management).
``` bash
opkg update
```

## Setup
### Replace Dropbear with OpenSSH
Dropbear doesn't work with  openssh-sftp-server, so we will replace it with OpenSSH to create a multi-user SFTP server.
* Via LuCI, change Dropbear port to something other than 22 (I used 44)
* Connect to router via Dropbear SSH on the new port
* Install OpenSSH (and SFTP)
``` bash
opkg install openssh-server openssh-sftp-server
```
* Enable & start OpenSSH
``` bash
/etc/init.d/sshd enable
```
``` bash
/etc/init.d/sshd start
```
* Log out of Dropbear SSH and reconnect with OpenSSH (using port 22 now)
* Stop & disable Dropbear
``` bash
/etc/init.d/dropbear disable
```
``` bash
/etc/init.d/dropbear stop
```
* Via LuCI, delete the Dropbear instance

### Users and groups
#### Add OpenWRT users
* Install necessary packages to manage users and groups
``` bash
opkg install shadow-useradd shadow-groupadd shadow-userdel shadow-groupdel shadow-groupmod shadow-usermod
```
* Add users, making sure to add a user to be designated as an administrator to use instead of the root user **(repeat for each user)**
``` bash
useradd [username]
```
* Add home directory
``` bash
mkdir /home
```
* Add user home directories **(repeat for each user)**
``` bash
mkdir /home/[username]
```
* Set user passwords - this is necessary because we won't be giving SFTP users shell access **(repeat for each user)**
``` bash
passwd [username]
```

#### Configure shell access for new users
* Add shell access for the administrative user created above
``` bash
usermod -s /bin/ash [admin_username]
```
* Disable shell access for the SFTP users created above **(repeat for each user)**
``` bash
usermod -s /bin/false [sftp_username]
```

#### Create groups to manage access rights
* Create an administrator group (this will be assigned to the administrative user created above)
``` bash
groupadd [admin_group_name]
```
* Create a SFTP  group (this will be the read-only SFTP-access-only user group)
``` bash
groupadd [sftp_group_name]
```

#### Assign groups to users
* Assign administrator group to the administrative user
``` bash
usermod -G [admin_group_name] [admin_username]
```
* Assign SFTP group to SFTP users **(repeat for each user)**
``` bash
usermod -G [sftp_group_name] [sftp_username]
```

#### Secure SSH access
* Install sudo to allow temporary privileged access
``` bash
opkg install sudo
```
* Edit ``` /etc/sudoers ``` to let any user 'sudo' with root password
``` bash
visudo
```
  * Uncomment the two lines below
  ``` bash
  Defaults targetpw  # Ask for the password of the target user
  ALL ALL=(ALL) ALL  # WARNING: only use this together with 'Defaults targetpw'
  ```
* Save and close ``` /etc/sudoers ```
* Edit SSH configuration file
``` bash
vim /etc/ssh/sshd_config
```
  * Set SSH & SFTP port
  ``` bash
  Port [your_desired_port]
  ```
  * Disable root SSH access
  ``` bash
  PermitRootLogin no
  ```
  * Change SFTP server from ``` Subsystem sftp /usr/lib/sftp-server ``` to:
  ``` bash
  Subsystem sftp internal-sftp
  ```
  * Configure chroot for SFTP group
  ``` bash
  Match Group [sftp_group_name]
          ForceCommand internal-sftp
          ChrootDirectory %h
          AllowTcpForwarding no
          PermitTunnel no
          X11Forwarding no
          AllowAgentForwarding no
  ```
* Save and close SSH configuration file
* Restart SSH
``` bash
/etc/init.d/sshd restart
```

**_At this point you can test SSH and SFTP access for the administrator and other users. Root SSH access is now disabled (use ``` sudo -i ``` as the administrative user to gain sudo privileges) Later on we'll be adding mount points so there's actually a point for SFTP-only users to access the router._**

### Storage
#### Connect hardware
Connect your EXT4 USB HDD to the router and if applicable, connect your SWAP formatted USB flash drive. The EXT4 HDD will be used as a network drive to store BitTorrent data in addition to Transmission configuration files.

#### Install necessary packages
* To read/write EXT4 filesystems
``` bash
opkg install kmod-fs-ext4
```
* To manage drives, including SWAP
``` bash
opkg install block-mount
```

#### Configure fstab
* Pre-fill fstab with necessary device information
``` bash
block detect > /etc/config/fstab
```
* Edit fstab
``` bash
vim /etc/config/fstab
```
  * Enable network drive HDD and SWAP (add option to both mount configs)
  ``` bash
  option enabled '1'
  ```
  * Enable read/write for network drive HDD
  ``` bash
  option options 'rw'
  ```
  * For the network drive HDD, change target to your naming preference
  ``` bash
  option target '/mnt/[network_drive]'
  ```
* Save and close fstab

#### Grant users directory access
* Within each SFTP user's home directory at ``` /home/[username] ```, create top-level directories mirroring ``` /mnt/[network_drive]/[shared_directory] ``` **(repeat for each user and shared directory)**
``` bash
mkdir /home/[username]/[shared_directory]
```
* Link ``` /home/[username]/[shared_directory] ``` to ``` /mnt/[network_drive]/[shared_directory] ``` on router startup
  * Edit router startup file
  ``` bash
  vim /etc/rc.local
  ```
  * Add to ``` /etc/rc.local ``` before the ``` exit 0 ``` line **(repeat for each user and shared directory)**
  ``` bash
  # Mounts network drive for SFTP users (remember to create /home/[username]/[shared_directory] first)
  mount --bind /mnt/[network_drive]/[shared_directory]/ /home/[username]/[shared_directory]/
  ```
* Restart router

**_At this point users can (locally) login and access directories on the network drive you gave them access to. Permissions will be set in the next section._**

### Permissions
We need to ensure permissions are set correctly on the network drive so that the SFTP group cannot write anything to the drive. We should also give the administrator group write access.
* Start by ensuring everything on the drive is initially owned by root
``` bash
chown -R root:root /mnt/[network_drive]
chmod -R 755 /mnt/[network_drive]
```

**_At this point, the SFTP and administrator groups only have read access to the shared directories._**

* Assign the network drive to the administrator group
``` bash
chgrp -R [admin_group_name] /mnt/[network_drive]
```
* Grant write access to the administrator group
``` bash
chmod -R 775 /mnt/[network_drive]
```
* Now, we need to ensure the administrator group stays assigned for all new files and directories. We make it 'stick' with the sticky bit.
``` bash
chmod -R g+s /mnt/[network_drive]
```

**_At this point permissions have been correctly configured. After configuring DDNS, your users will be able to safely access your shared directories._**

### DDNS
There are many different DDNS options available. Follow the guide [here](http://wiki.openwrt.org/doc/howto/ddns.client) to choose and configure a DDNS client that best meets your needs.

> ###### NOTE: Add a firewall exception for SSH/SFTP remote access
> ![Example of firewall exception in /etc/config/firewall](http://i.imgur.com/OqLvVoO.png)

**_At this point your users will be able to access your shared directories remotely by using your DDNS address and SSH/SFTP port._**

### Samba (optional)
If you're on a Windows-based system, you'll want to access your network drive via Windows File Explorer.
* Install Samba
``` bash
opkg install luci-app-samba
```
* Configure Samba through LuCI (settings are pretty self-explanatory)
![Example of LuCI Samba configuration](http://i.imgur.com/uXtFGBP.png)
* Edit ``` /etc/samba/smb.conf.template ```
``` bash
vim /etc/samba/smb.conf.template
```
  * Add the following lines to ``` /etc/samba/smb.conf.template ```
  ``` bash
  force create mode = 0775
  force directory mode = 0775
  ```
  * Remove or comment the following line from ``` /etc/samba/smb.conf.template ``` (we will be using root to access the drive via your local network)
  ``` bash
  invalid users = root
  ```
* Add Samba user(s) (they must exist in ``` /etc/passwd ``` - I used the root user to keep root ownership of files and directories)
``` bash
smbpasswd -a [username]
```
> ###### NOTE: The password you set here is different than the SSH/SFTP password set for the user.

**_At this point you can mount your network drive locally in Windows via the "Map Network Drive" feature._**

### Transmission
* Install necessary Transmission packages
``` bash
opkg install luci-app-transmission transmission-web
```
* Configure Transmission via LuCI to preferred settings (most will depend on your ISP speed)
  * Point configuration directory to the network drive
  ``` bash
  config file directory = /mnt/[network_drive]/transmission
  ```
  * Set the network drive as the download directory
  ``` bash
  download directory = /mnt/[network_drive]/[specific_directory]
  ```
  * Set umask to 2 (equates to 002) so the correct permissions (775) are set on new files and directories
  ``` bash
  umask = 2
  ```

Make sure you configure a RPC username and password for Transmission. Once secured, you can disable the RPC whitelist to enable remote Transmission access (you'll still need to configure a firewall exception for Transmission web interface access).

## Wrap-up
And that's all folks. You can view Transmission web interface documentation [here](https://trac.transmissionbt.com/wiki/WebInterface). The number of BitTorrent transfers possible will depend on your specific router and OpenWRT release stability.

I'll add additional information (and potentially some media of the finished product) in the future. Questions or found a bug? Email me at [me@matthewtraughber.com](mailto:me@matthewtraughber.com) or create an issue or pull request.
