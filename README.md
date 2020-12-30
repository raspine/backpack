# backpack
A recepy for makings backups using a RPi and an external USB HDD (and tejp..).

## Motivation
Since making backups manually is extremely boring and easy to forgett,
backups needs to be simple to do manually and/or automated.

[rsync](https://rsync.samba.org/) is our champion that perfectly aligns
directories and files between two HDD:s, however most guides I found only cover
the rsync part and I wanted a more complete instruction covering:

* system installation from scratch with typical config steps
* long term operation such as avoid wearing out the RPi SD card
* S.M.A.R.T monitoring of the HDD with e-mail notification
* rsync with e-mail notification
* both automatic (on-line) backups and manual (off-line) backups schemes

This guide uses a step-by-step approach so nothing is forgotten. For details
about the various steps, refer to the links in the Arch wiki.

## Install Arch Linux

[Arch Linux ARM](https://archlinuxarm.org) provides a pretty small initial
footprint so that's my starting point.

Prefer to use a big SD card >=32GB as this better avoids wearing out over time.

When the RPi boots up, log in as root (ssh server is enabled by default) and
continue with this instruction.

### Update Arch Linux

([Arch wiki ref](https://wiki.archlinux.org/index.php/pacman))

System update:

```bash
# pacman -Syu
```

```bash
# reboot
```
When the RPi boots up, log in as root and continue with this instruction.

## Manage accounts

([Arch wiki ref](https://wiki.archlinux.org/index.php/users_and_groups))
([Arch wiki ref](https://wiki.archlinux.org/index.php/sudo))

Change root password:

```bash
# passwd
```

Remove default user account and add your own:

```bash
# userdel alarm
# useradd -m -G wheel -s /bin/bash <username>
# passwd <username>
```

Install `sudo` and allow for passwordless operation.
```bash
# pacman -S sudo
# visudo
## uncomment line '%wheel ALL=(ALL) NOPASSWD: ALL'
```

## Set keyboard layout

([Arch wiki ref](https://wiki.archlinux.org/index.php/Keyboard_configuration_in_console))

List all keyboard layouts:

```bash
# localectl list-keymaps
```

Load immediately:

```bash
# loadkeys uk
```

Make persistent:

```bash
# echo "KEYMAP=uk" > /etc/vconsole.conf
```

## Configure network
([Arch wiki ref](https://wiki.archlinux.org/index.php/systemd-networkd))

For simplicity I prefer using static IP for these types of machines.

Edit the `[Network]` section in /etc/systemd/network/eth0.network:
It should look something like this:

```
[Match]
Name=eth0

[Network]
Address=192.168.0.XXX/24
Gateway=192.168.0.1
```

Make it effective by restarting the networkd service:

```bash
# systemctl restart systemd-networkd
```
## Set hostname

```bash
# hostnamectl set-hostname <backpackX>
```

## Configure timezone
([Arch wiki ref](https://wiki.archlinux.org/index.php/Time$Time_zone))

Find the matching time zone in /usr/share/zoneinfo/\*/\*. Then:

```bash
# rm /etc/localtime
# ln -s /usr/share/zoneinfo/Europe/Stockholm /etc/localtime
```

## NTP
([Arch wiki ref](https://wiki.archlinux.org/index.php/systemd-timesyncd))

Arch should already be set up with NTP sync so nothing to do here.

## Extend life time of SD card
This [thread](https://raspberrypi.stackexchange.com/questions/169/how-can-i-extend-the-life-of-my-sd-card)
has a lot of tips.
I ended up doing the following:

### Disable swap
([Arch wiki ref](https://wiki.archlinux.org/index.php/swap))

Arch does not enable swap by default so nothing to do here. Check with:

```bash
# free -h
```

### tmpfs
([Arch wiki ref](https://wiki.archlinux.org/index.php/tmpfs))

Use tmpfs for various locations instead of writing to SD card.

```bash
# echo "tmpfs   /var/log    tmpfs    defaults,noatime,nosuid,mode=0755,size=40m    0 0" >> /etc/fstab
# echo "tmpfs   /var/tmp    tmpfs    defaults,noatime,nosuid,size=20m    0 0" >> /etc/fstab
# echo "tmpfs   /tmp        tmpfs    defaults,noatime,nosuid,size=40m    0 0" >> /etc/fstab
```

### systemd journal
([Arch wiki ref](https://wiki.archlinux.org/index.php/systemd$Journal))

When using tmpfs the /var/log/journal directory will not be re-created and
journald will instead write to /run/systemd/journal in a non-persistent way.
This seems to be the preferred way so no configuration is needed.
We can also limit the journal size:
```bash
# mkdir /etc/systemd/journald.conf.d
# echo -e "[Journal]
SystemMaxUse=50M" > /etc/systemd/journald.conf.d/00-journal-size.conf
```

## Configure email notification
([Arch wiki ref](https://wiki.archlinux.org/index.php/Msmtp))

Followed the basic setup in the Arch wiki and installed.
```bash
# pacman -S msmtp msmtp-mta s-nail
```

As I use two factor authentication for my Google account I generated an app
password and used in the ~/.msmtprc password field.

Test with:

```bash
$ echo "body" | mail -s "headline" <username@gmail.com>
```

## S.M.A.R.T.
([Arch wiki ref](https://wiki.archlinux.org/index.php/S.M.A.R.T.))

Lots of info in the Arch wiki. However setup is easy.

```bash
# pacman -S smartmontools
```

Check that the external harddrives has S.M.A.R.T. capabilities:

```bash
# smartctl --info /dev/sda
```

**Note! Comment the default DEVICESCAN line in /etc/smartd.conf**

To test e-mail notification:

```bash
# echo "DEVICESCAN -m <username@gmail.com> -M test" >> /etc/smard.conf
# systemctl restart smartd
```

For the actual S.M.A.R.T. monitor settings I use the suggested line in the Arch
wiki, however increased the upper temperature from 40 to 45 as I otherwise got
daily temperature alarms and 45 seems to be an acceptable upper limit.

*Remember to uncomment previous email test line*

```bash
# echo "DEVICESCAN -a -o on -S on -n standby,q -s (S/../.././02|L/../../6/03) -W 4,35,45 -m <username@gmail.com>" >> /etc/smartd.conf
# systemctl enable smartd
```

## Prepare external HDD

([Arch wiki ref](https://wiki.archlinux.org/index.php/NTFS-3G))

For maximum portability I use Windows NTFS as file system for the external HDD.
Another option would be FAT32, but it can't handle file sizes larger than 4GB.

For our backup we need to write to NTFS partitions so install ntfs-3g:

```bash
# pacman -S ntfs-3g
```

Make sure the external HDD is plugged in and check its fs type (should be
listed as /dev/sda1)

```bash
# lsblk -f
```

In case the HDD needs formating:

```bash
# fdisk /dev/sdX
## run commands 'o', 'n' <accept all defaults>, 't' <07>, 'w'
# mkfs.ntfs -Q -L <diskLabel> /dev/sdX1
```

## Mounting

([Arch wiki ref](https://wiki.archlinux.org/index.php/fstab))

Create mount points (for every source/backup directory):

```bash
# mkdir -p /mnt/exthdd/<shared_dir1>
# mkdir -p /mnt/<shared_dir1>
# chown -R <username>:<username> /mnt/<shared_dir1>
```

As my NAS provides NFS shares, install nfs-utils:

([Arch wiki ref](https://wiki.archlinux.org/index.php/NFS))

```bash
# pacman -S nfs-utils
```

Mount the network shared directories (backup from) and the external HDD (backup to)
in /etc/fstab:

```
...

/dev/sda1  /mnt/exthdd  ntfs  defaults  0  0
<NAS ip address>:/srv/nfs4/<shared_dir1>  /mnt/<shared_dir1>  nfs  noauto,vers=3,x-systemd.automount,x-systemd.device-timeout=10,timeo=14,x-systemd.idle-timeout=1min  0  0 
```

Reboot and check that mount points work:
```bash
$ reboot
$ ls /mnt/exthdd
$ ls /mnt/<shared_dir1>
```

## rsync

([Arch wiki ref](https://wiki.archlinux.org/index.php/rsync))

rsync is used to copy file from shared network directories to the external HDD.
From now on login as normal user and use `sudo` where needed.

```bash
$ sudo pacman -S rsync
```

We put the rsync commands in a bash script along with some other lines:
```bash
$ touch /home/<username>/rsync_nas.sh
$ chnmod +x /home/<username>/rsync_nas.sh
```

Edit rsync_nas.sh as follows:

```
#!/bin/bash
LOG_FILE=/var/log/backpack.log
### EDIT THESE ###
USER_NAME=<username>
EMAIL=<mail@address.com>
declare -a SRC_DIRS=("/mnt/shared_dir1" "/mnt/shared_dir2")
declare -a DST_DIRS=("/mnt/exthdd/shared_dir1" "/mnt/exthdd/shared_dir2")

# create empty log file so it only include the last rsync log
rm $LOG_FILE &> /dev/null
sudo touch $LOG_FILE
sudo chown $USER_NAME:$USER_NAME $LOG_FILE

# do rsync for each directory
for (( i=0; i<${#SRC_DIRS[@]}; i++ ));
do
  echo -e "rsync ${SRC_DIRS[$i]}:\n===========================================================================\n" >> $LOG_FILE
  rsync -av --delete --log-file=$LOG_FILE ${SRC_DIRS[$i]} ${DEST_DIRS[$i]} &> /dev/null
done

# mail the log file
ERR=`cat $LOG_FILE | mail -s "$HOSTNAME rsync result" "$EMAIL"`
[ ! -z "$ERR" ] && echo "$ERR" | mail -s "$HOSTNAME rsync error" "$EMAIL"
```

## Backup schemes
Two types of backups are used. I refer to them as on-line backups and off-line
backups.
on-line backups sits on the same network and are always powered up. These
backups use systemd (see below) to schedule backup cycles.

Off-line backups are not powered up and located elsewhere. I bring these
backups to my home e.g. two times per year, connect the USB cable to the RPi
and just run rsync_nas.sh manually.

### Automatic/scheduled backups
([Arch wiki ref](https://wiki.archlinux.org/index.php/Systemd/Timers))
Create a systemd service that runs rsync_nas.sh.
```bash
# echo -e "[Unit]
Description=rsync job

[Service]
Type=oneshot
ExecStart=/bin/bash /home/<username>/rsync_nas.sh" > /etc/systemd/system/rsync_nas.service
```

Create a systemd timer that runs above service at regular intervals.
```bash
# echo -e "[Unit]
Description=Run rsync_nas first Saturday every month

[Timer]
OnCalender=Sat *-*-1..7 05:00:00
Persistent=true

[Install]
WantedBy=timers.target" > /etc/systemd/system/rsync_nas.timer
```

Enable/start timer service.
```bash
# systemctl daemon-reload
# systemctl enable rsync_nas.timer
# systemctl start rsync_nas.timer
```

