# debian-sever-setup

#### Operating system (Debian 8 Jessie) install

Steps followed to install Debian 8 Jessie (stable) and base software on a server. Workstation is a Puget Systems Obsidian, with 32 GB RAM, 500 GB SSD, and Intel Xeon 3.6 GHz Quad-core.

A [net install .iso](https://www.debian.org/CD/netinst/) of Debain was downloaded for amd64, and put on a USB key using [Linux Live USB creator](http://www.linuxliveusb.com/). A graphical install was used. Root password, and user name and password are entered. Default settings were used in the install, with the following exceptions:
* Guided partitioning, with Logical volume management enabled
* Software selection: GNOME desktop env., web server, print server, SSH server, standard system utilities
* Installed GRUB boot loader to Master boot record (MBR)

#### User setup and management

After logging into Debian for the first time, start up a terminal. First we need to give `sudo` (root privileges) to our user we created in the install process (e.g., `user1`):

Switch to root user:

    su root
Install sudo, and enable sudo access for `user1`:

    apt-get update
    apt-get install sudo
    usermod -a -G sudo user1

Now switch back to `user1` and create any new users using the `adduser` command:

    su user1
    sudo adduser user2

To delete a user, check if they are logged in first (using `who`), then enter the following:

    sudo deluser --remove-home user2

We now can access the server remotely using `ssh user1@computer_name`. For security, we can disallow remote logins as the `root` user, by modifying the `/etc/ssh/sshd_config` file:

    sudo nano /etc/ssh/sshd_config
Change the line `#PermitRootLogin yes` to `PermitRootLogin no`, and then restart ssh:

    systemctl restart ssh

#### Set up networking

Install necessary packages for mounting network folders:

    sudo apt-get install samba
    sudo apt-get install smbclient
    sudo apt-get install cifs-utils

Make new local directories to link to network folders:

    sudo mkdir /mnt/basille_lab
    sudo mkdir /mnt/dbucklin

Since network folders require authentication, create a credentials text file with the following lines:

    username=*******
    password=*******
    domain=ad.ufl.edu

Network folders can be mounted using the following commands:

    sudo mount.cifs //ifs-flrec-1mps/data/Users/dbucklin /mnt/dbucklin -o credentials=/path/to/file,uid=user1,gid=user1
    sudo mount.cifs //ifs-flrec-1mps/data/Groups/basille_lab /mnt/basille_lab -o credentials=credentials=/path/to/file,uid=user1,gid=user1

To load network folder for the lab on computer startup, add the following line to `/etc/fstab`:

    //ifs-flrec-1mps/data/Groups/basille_lab /mnt/basille_lab cifs credentials=/path/to/file,uid=user1 0 0

#### Install PostgreSQL and related software

Install the base server, client, and development files - more information can be found [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-9-4-on-debian-8):

    sudo apt-get update
    sudo apt-get install postgresql-9.4 postgresql-client-9.4 postgresql-server-dev-9.4

The install creates a new system user `postgres`. This user can create additional database users (roles):

    su root
    su postgres
    createuser --interactive

Install PostGIS and pgAdmin3:

    sudo apt-get install pgadmin3
    sudo apt-get install postgis

Databases can then be imported using the pgAdmin3 restore tool (Tools->Restore). Make sure prior to restore that all roles who have privileges on the restored databases already exist on the server.

##### *Set up automatic backup to network for server databases*

First create a new folder in the lab network folder:

    sudo mkdir /mnt/basille_lab/db_backups

Now open `cron`, a task scheduling file, using the following command:

    crontab -e

Add the following lines to the file, creating a backup for each database as well as the entire server, and deleting old files in the backup folder:

    #backup databases, with dates in filenames
    pg_dump wood_stork_tracking | gzip > /mnt/basille_lab/db_backups/wood_stork_tracking_`date +'%Y_%m_%d'`.gz
    pg_dump keys_gps_tracking | gzip > /mnt/basille_lab/db_backups/keys_gps_tracking_`date +'%Y_%m_%d'`.gz
    #backup full database server
    pg_dumpall | gzip > /mnt/basille_lab/db_backups/fullDB_`date +'%Y_%m'`.gz
    #delete files older than 60 days
    find /mnt/basille_lab/db_backups -type f -mtime +60 -delete

##### *PostgreSQL setup and maintenence*

The main settings for the Postgresql server can be altered by editing `postgresql.conf`, and connection settings in `pg_hba.conf`:

    sudo nano /etc/postgresql/9.4/main/postgresql.conf
    sudo nano /etc/postgresql/9.4/main/pg_hba.conf

Following changes, restart the server using:

    sudo pg_ctlcluster 9.4 main [status][reload][restart][start][stop]

#### Virtual network computing (remote desktop)

Install vnc4server and the xfce4 desktop environment (there are issues with GNOME and Debian 8 on VNC):

    sudo apt-get install vnc4server
    sudo apt-get install xfce4 xfce4-goodies

Modify the file `/home/user1/.vnc/xstartup`:

    sudo nano /home/user1/.vnc/xstartup
    ##full file below
    #!/bin/sh
    unset SESSION_MANAGER
    unset DBUS_SESSION_BUS_ADDRESS
    startxfce4 &
    
    [ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
    [ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
    xsetroot -solid grey
    vncconfig -iconic &
    ####
