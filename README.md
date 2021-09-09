# 💽 backup-peertube

> Backup workflow of PeerTube docker stack with rsync or rsnapshot

### Prerequisites

Before any backup **your have to export the PeerTube database**, see [install-peertube documentation guide](https://github.com/kimsible/install-peertube/blob/master/DOCUMENTATION.md#backup-and-restore).

### Mirror Backup

The mirror backup may be between your production server and a remote cloud / FTP.

Production server requirements: `rsync` and `sshpass`.

Backup command

```bash
$ rsync -av --delete --ignore-times --exclude docker-volume/db /var/peertube/ username@remote-cloud:<backups-absolute-path>/var/peertube/
```

To include the backup command in a script without password prompt, you might use sshpass before :
```bash
$ sshpass -p yourpassword rsync -av --exclude docker-volume/db /var/peertube/ username@remote-cloud:<backups-absolute-path>/var/peertube/ --delete
```

#### Basic Restore command only for missing files

```bash
$ rsync -av --delete username@remote-cloud:<backups-absolute-path>/var/peertube/ /var/peertube/
```

#### Full Restore command

```bash
$ rsync -av --delete --ignore-times username@remote-cloud:<backups-absolute-path>/var/peertube /var/peertube
```

For specific protocol without rsync or SSH support like FTP, you might use [rclone](https://rclone.org/ftp/).

#### Crontab

- Edit crontab with `crontab -u root -e`
- Add this line to run it as docker user every day at 5:25am :

```bash
25 5  * * * /usr/bin/peertube-mirror
```

### Incremential Backup

The incremential backup maybe between a local OrangePi / RaspberryPi with a connected external disk and the remote production server.

Local OPI/ RPI requirements : `rsync`, `open-ssh` and  `rsnapshot`.

#### **Operating system - fstab**

First of all your need to configure :

- OPI / RPI connected to internet and with a local SSH access;

- External disk to `fstab` mounted on a home sub-directory :

Get UUID of your disk:
```bash
$ blkid
# /dev/sda1: UUID="7610ebc5-5231-4b59-830e-a9sacb84a" TYPE="ext4" PARTUUID="62a1e04b-b700-fc4e-5878-78863daf9b34"
```
Insert the line bellow into `/etc/fstab`
```
UUID=<UUID> <mounted-disk-absolute-path> auto    rw,user,auto    0    0
```


#### **SSH keys**

On local OPI / RPI, generate SSH keys:

```bash
$ ssh-keygen -t rsa
```

At last copy generated SSH keys from local OPI / RPI to the remote production server:
```bash
$ ssh-copy-id <user>@<remote-production>
```


#### **Rsnapshot config**
On local OPI / RPI, you need to configure rsnapshot.

Before, you may backup the original configuration:

```bash
$ mv /etc/rsnapshot.conf /etc/rsnapshot.conf.backup
```

And put this configuration into `/etc/rsnapshot.conf` :

_Carefull, the space between args are tabs_
```
config_version  1.2
snapshot_root   <mounted-disk-absolute-path>
cmd_cp    /bin/cp
cmd_rm    /bin/rm
cmd_rsync   /usr/bin/rsync
cmd_ssh   /usr/bin/ssh
cmd_logger    /usr/bin/logger
cmd_du    /usr/bin/du
cmd_rsnapshot_diff    /usr/bin/rsnapshot-diff
interval    hourly    6
interval    daily     7
interval    weekly    4
interval    monthly   3
verbose     2
loglevel    4
lockfile    /var/run/rsnapshot.pid
use_lazy_deletes    1
rsync_numtries      2
rsync_short_args        -a
rsync_long_args --delete --numeric-ids --relative --delete-excluded
backup_exec     ssh <user>@<remote-production> "/usr/sbin/peertube postgres:dump /var/peertube/docker-volume/db.tar"
backup  <user>@<remote-production>:/var/peertube  . exclude=docker-volume/db
```

Test this configuration with
```bash
$ rsnapshot configtest
```

#### **Backup**
```bash
$ rsnapshot daily
```

#### **Basic Restore command only for missing files**
```bash
$ rsync -av --delete <mounted-disk-absolute-path>/daily.<day>/var/pertube/  <user>@<remote-production>:/var/pertube/
```

#### **Full Restore command**

```bash
$ rsync -av --delete --ignore-times <mounted-disk-absolute-path>/daily.<day>/var/peertube/  <user>@<remote-production>:/var/pertube/
```

#### **Crontab**
On OPI / RPI:

- Edit crontab with `crontab -u root -e`
- Add this line to run it as docker user every day at 4:25am :

```bash
25 4  * * * /usr/bin/rsnapshot daily
```



