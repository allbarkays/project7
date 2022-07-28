# Project 7
## Step 1 – Prepare NFS Server

* Spin up an EC2 instance with Red Hart OS as NFS server

* Follow up by adding 2 volumes in the same AZ as the NFS Server, each with 8 GB storage space

* Then used ***lsblk*** command to inspect the block devices that are attached to my NFS server

`lsblk`


![lsblk.PNG](./images/lsblk.PNG)


* Used the ***gdisk utility*** to create a single partition on each of the disks ***xvdb, xvdc***

`sudo gdisk /dev/xvdb`

`sudo gdisk /dev/xvdc`


![partition-xvdb1-xvdc1.PNG](./images/partition-xvdb1-xvdc1.PNG)


* Installed ***lvm2*** package

`sudo yum install lvm2`


* Then checked for available partitions

`sudo lvmdiskscan`


![sudo-lvmdiskscan.PNG](./images/sudo-lvmdiskscan.PNG)


* With `sudo pvcreate /dev/xvdb1 /dev/xvdc1` command, set up the two blocks as physical volumes for LVM

* And the verify with `sudo pvs`


![sudo-pvs.PNG](./images/sudo-pvs.PNG)


* created nfs volume group and verified

`sudo vgcreate nfs-vg /dev/xvdb1 /dev/xvdc1`


`sudo vgdisplay`



![nfs-vgs.PNG](./images/nfs-vgs.PNG)

Using lvcreate utility to create 3 logical volumes ***lv-opt , lv-apps and lv-logs*** with an allocation of 5 GB on each

`sudo lvcreate -n lv-apps -L 5G nfs-vg`

`sudo lvcreate -n lv-opt -L 5G nfs-vg` 

`sudo lvcreate -n lv-logs -L 5G nfs-vg`

* Then verified with:

`sudo lvs`

![sudo-lvs.PNG](./images/sudo-lvs.PNG)

* As in the documentation, I used ***mkfs.xfs*** to format the logical volumes

`sudo mkfs -t xfs /dev/nfs-vg/lv-opt`


`sudo mkfs -t xfs /dev/nfs-vg/lv-apps`


`sudo mkfs -t xfs /dev/nfs-vg/lv-logs`

* Created mount points on ***/mnt directory*** for the logical volumes as follow:


`sudo mkdir /mnt/apps /mnt/logs /mnt/opt`



-- Mount lv-apps on /mnt/apps – ***To be used by webservers***

-- Mount lv-logs on /mnt/logs – ***To be used by webserver logs***

-- Mount lv-opt on /mnt/opt – ***To be used by Jenkins server***


`sudo mount /dev/nfs-vg/lv-apps /mnt/apps`


`sudo mount /dev/nfs-vg/lv-logs /mnt/logs`


`sudo mount /dev/nfs-vg/lv-opt /mnt/opt`


* Update ***/etc/fstab*** file so that the mount for consistency after server restart 

1. run: `sudo blkid /dev/nfs-vg/*`

2. Copy the UUID of the device to update in the /etc/fstab file

3. `sudo vi /etc/fstab`

4. Update /etc/fstab by pasting the UUID copied above.



Did `sudo less /etc/fstab` to see added UUID

![sudo-less-etc-fstab.PNG](./images/sudo-less-etc-fstab.PNG)


* Test the configuration and reload

`sudo mount -a`

`sudo systemctl daemon-reload`

Confirmed setup with: 
`df -h`


![df-h.PNG](./images/df-h.PNG)


* Install NFS server, configure it to start on reboot and make sure it is u and running


`sudo yum -y update`

`sudo yum install nfs-utils -y`

`sudo systemctl start nfs-server.service`

`sudo systemctl enable nfs-server.service`

`sudo systemctl status nfs-server.service`


***NFS server running active***

![nfs-running-active.PNG](./images/nfs-running-active.PNG)


* Set up permission that will allow our Web servers to read, write and execute files on NFS:

`sudo chown -R nobody: /mnt/apps`

`sudo chown -R nobody: /mnt/logs`

`sudo chown -R nobody: /mnt/opt`

* Read, write, execute permission

`sudo chmod -R 777 /mnt/apps`

`sudo chmod -R 777 /mnt/logs`

`sudo chmod -R 777 /mnt/opt`

* Restart NFS server

`sudo systemctl restart nfs-server.service`


* Next, configure access to NFS for clients within the same subnet




* Export the mounts for my webservers’ subnet CIDR (172.31.16.0/20) to connect as clients.

* Configure access to NFS for clients within the same subnet. using vi editor,input my subnet IPv4 cidr.

`sudo vi /etc/exports`

/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)


![sudo-less-etc-exports.PNG](./images/sudo-less-etc-exports.PNG)


`sudo exportfs -arv`

![exports.PNG](./images/exports.PNG)


* Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)

`rpcinfo -p | grep nfs`

![npcnfs-info.PNG](./images/npcnfs-info.PNG)


*  In order for NFS server to be accessible from clients, I updated the inbound rule to open following ports: ***TCP 111, UDP 111, UDP 2049, TCP 2049***



![update-inbound-rule.PNG](./images/update-inbound-rule.PNG)



## Step 2 - Configure the database server
By now you should know how to install and configure a MySQL DBMS to work with remote Web Server

* Install ***MySQL server***
* Create a database and name it ***tooling***
* Create a database user and name it ***webaccess***
* Grant permission to ***webaccess*** user on ***tooling*** database to do anything only from the webservers ***subnet cidr***

`sudo mysql`

`sudo systemctl status mysql.service`

![mysql-running.PNG](./images/mysql-running.PNG)

```

CREATE DATABASE tooling;

CREATE USER 'webaccess'@'%' Identified with mysql_Native_password BY 'password';

GRANT ALL ON tooling.* TO 'webaccess'@'%';

FLUSH PRIVILEGES;

SHOW DATABASES;

SELECT USER FROM mysql.user;

exit

```

![database-user-created.PNG](./images/database-user-created.PNG)


* Updated bind-address

`sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf`

![bind-address.PNG](./images/bind-address.PNG)


* Restart mysql

`sudo systemctl restart mysql`


## Setting up web-servers 


I launched three new EC2 instance with RHEL 8 Operating System

Then, installed NFS client on each

`sudo yum install nfs-utils nfs4-acl-tools -y`


After which I attempted to mount ***/var/www/*** and target the NFS server’s export for apps

`sudo mkdir /var/www`

`sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www`

i.e 
`sudo mount -t nfs -o rw,nosuid 172.31.17.239:/mnt/apps /var/www`


but it failed to mount as I got the error below:

```
mount.nfs: access denied by server while mounting 172.31.17.239:/mnt/apps
```















































