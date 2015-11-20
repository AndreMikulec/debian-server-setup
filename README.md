# debian-sever-setup

#### Operating system (Debian 8 Jessie) install

Steps followed to install Debian 8 Jessie (stable) and base software on a server. Workstation is a Puget Systems Obsidian, with 32 GB RAM, 500 GB SSD, and Intel Xeon 3.6 GHz Quad-core.

A [net install .iso](https://www.debian.org/CD/netinst/) of Debain was downloaded for amd64, and put on a USB key using [Linux Live USB creator](http://www.linuxliveusb.com/). A graphical install was used. A root password, and a new user name and password are entered. Default settings were used in the install, with the following exceptions:
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

##### *Install PostgreSQL 9.4*

Install the base server, client, and development files - more information can be found [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-9-4-on-debian-8):

    sudo apt-get update
    sudo apt-get install postgresql-9.4 postgresql-client-9.4 postgresql-server-dev-9.4

The install creates a new system user `postgres`. This user can create additional database users (roles), using `createuser` and answering the questions that follow:

    su root
    su postgres
    createuser --interactive
    Enter name of role to add: user1
    Shall the new role be a superuser? (y/n) y
    Shall the new role be allowed to create databases? (y/n) y
    Shall the new role be allowed to create more new roles? (y/n) y
    
Now we can log into `psql` as `user1` using the following command:

    psql -d database_name

##### *Install PostGIS and pgAdmin3*

    sudo apt-get install pgadmin3
    sudo apt-get install postgis

Databases can then be imported using the pgAdmin3 restore tool (Tools->Restore). Make sure prior to restore that all roles who have privileges on the restored databases already exist on the server. You could also restore the database using [psql](http://www.postgresql.org/docs/9.4/static/backup-dump.html).

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

Modify the file:

    sudo nano /home/user1/.vnc/xstartup

Full file below:

    #!/bin/sh
    unset SESSION_MANAGER
    unset DBUS_SESSION_BUS_ADDRESS
    startxfce4 &
 
    [ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
    [ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
    xsetroot -solid grey
    vncconfig -iconic &
    
Now launch a vnc server using, and note the name and number (e.g., `computer-name:1`) given to it:

    vnc4server -geometry 1920x1080 -depth 24
    
To stop the server `computer-name:1`:

    vnc4server -kill :1

#### Install R and supporting programs

##### *R (current version)*

Install main packages:

    sudo apt-get update
    sudo apt-get install r-base r-base-dev
    sudo apt-get install libatlas3-base
    
By default (on Debian Jessie), R 3.1.1 is installed. To set up backports for Jessie to allow for updating R, add an appropriate mirror [source](https://cran.r-project.org/mirrors.html) to `/etc/apt/sources.list`:

    deb http://archive.linux.duke.edu/cran/bin/linux/debian jessie-cran3/

We also need to add a PUBKEY for the R mirror we chose:

    gpg --keyserver pgpkeys.mit.edu --recv-key 06F90DE5381BA480
    gpg -a --export 06F90DE5381BA480 | sudo apt-key add -
    
Now we can upgrade R to the newest version (as of Nov 2015, 3.2.2):

    sudo apt-get update
    sudo apt-get upgrade
    
To install R packages globally, we need to open R with root privileges (`sudo R`). Instead of doing this every time, we can add users to the group `staff`, which then allows those users to install to the global R library folder (`/usr/local/lib/R/site-library`) by default, e.g.:

    su root
    adduser user1 staff
    su user1
    R
    > install.packages('shiny')

##### *RStudio Server*

Download and install gdebi and RStudio Server 64-bit, and start it:

    sudo apt-get install gdebi-core
    wget https://download2.rstudio.org/rstudio-server-0.99.489-amd64.deb
    sudo gdebi rstudio-server-0.99.489-amd64.deb
    sudo rstudio-server start

Server can be accessed [here](http://basille-flrec.ad.ufl.edu:8787/auth-sign-in).

##### *Shiny server*

Before istalling the Shiny server in Debian 8, a prerequiste package (libssl0.9.8) needs to be installed:

    wget http://ftp.us.debian.org/debian/pool/main/o/openssl/libssl0.9.8_0.9.8o-4squeeze14_amd64.deb
    sudo dpkg -i libssl0.9.8_0.9.8o-4squeeze14_amd64.deb

Now install the Shiny server:

    wget https://download3.rstudio.org/ubuntu-12.04/x86_64/shiny-server-1.4.0.756-amd64.deb
    sudo gdebi shiny-server-1.4.0.756-amd64.deb

To share an app on the server, just upload it's project folder (containing `server.r` and `ui.r` to server folder, e.g.:

    sudo cp -R /usr/local/lib/R/site-library/shiny/examples/04_mpg /srv/shiny-server/

Apps are shared at `http://basille-flrec.ad.ufl.edu:3838/app_name`.

##### *System packages necessary for R packages*

Install gdal, and necessary packages for using the R package `rgdal`:

    sudo apt-get install gdal-bin
    sudo apt-get install libproj-dev
    sudo apt-get install libgdal-dev
    
Install necessary packages for the R package `devtools`:

    sudo apt-get install libssl-dev
    sudo apt-get install libxml2-dev
    sudo apt-get libcurl4-openssl-dev
    

    

