

# Implementing  WordPress website with Logical Volume Management (LVM) 

### Project Scope:

- Configure storage subsystem for Web and Database servers, we will work with disks, partitions, and volumes in Linux.

- Install WordPress and connect it to a remote My SQL database server.


##

### Three-tier Architecture

Generally, web or Mobile solutions are Implemented based on what is called Three-tier Architecture

**Presentation Layer(PL)**: This is the user interface such as the client-server or browser on your Laptops.

**Business Layer(BL)**: This is the backend program that implements business logic. Application or webserver.

**Data Access or Management Layer(DAL)**: layer for the computer data storage and data access. Database server or File system such as FTP server or NFs server.


##

1. Launch a RedHat EC2 instance, and create 3 volumes in the same AZ as your EC2 instance.



![image](images/Screenshot_3.png)

- Create a volume of 10Gib

![image](images/Screenshot_1.png)



![image](images/Screenshot_2.png)

2. Attach all three volumes one by one to your web server EC2 instance.

![image](images/Screenshot_4.png)

3. Use `lsblk` to inspect what block devices are attached to the server. Names of your newly created devices will show. 

   - All devices reside in `/dev/directory`, inspect it with `ls /dev/`

![image](images/Screenshot_5.png)

4. Use `df -h` to see all mounts and free space on your server

![image](images/Screenshot_6.png)

5. Use `gdisk` to create a single partition on each of the 3 disks.

   - `sudo gdisk /dev/xvdf`
   - `sudo gdisk /dev/xvdg`
   - `sudo gdisk /dev/xvdh`

![image](images/Screenshot_7.png)

6. Use `lsblk` to view the newly configured partitions.

![image](images/Screenshot_8.png)

7. Run `sudo yum install lvm2` to install the LMV2 package.  Then `sudo lvmdiskscan` to check for available partitions.

8. Use `pvcreate` to mark each of the 3 disks as physical volumes to be used by LVM.

   Run
    - `sudo pvcreate /dev/xvdf1`
    - `sudo pvcreate /dev/xvdg1`
    - `sudo pvcreate /dev/xvdh1`

![image](images/Screenshot_9.png)

![image](images/Screenshot_10.png)

![image](images/Screenshot_11.png)

9. Run `sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1` this will add all 3 PV.s to a volume group(VG). We have named the VG web-data.

![image](images/Screenshot_12.png)

10. Use `lvcreate` to create 2 logical volumes, one for storing data and the other for logs.

    - `sudo lvcreate -n app-lv -L 14G webdata-vg`
    - `sudo lvcreate -n logs-lv -L 14G webdata-vg`

![image](images/Screenshot_13.png)

11. Run `sudo vgdisplay -v` to verify the entire setup then run `lsblk`

![image](images/Screenshot_14.png)

![image](images/Screenshot_15.png)

12. Next format the logical volumes with **ext4** filesystem

    - Run `sudo mkfs -t ext4 /dev/webdata-vg/apps-lv`
    - Run `sudo mkfs -t ext4 /dev/webdata-vg/logs-lv`

13. Create **/var/www/html** dirctory to store the website files
    
    - `sudo mkdir -p /var/www/html`

14. Create **/home/recovery/logs** to store backup of log data

    - `sudo mkdir -p /home/recovery/logs`


![image](images/Screenshot_16.png)

15. Mount **/var/www/html** on **apps-lv** logical volume

    - Run `sudo mount /dev/webdata-vg/apps-lv /var/www/html`

16. - Run `sudo rsync -av /var/log/. /home/recovery/logs/` to backup all the files in the log dir **/var/log** into **/home/recovery/logs** then mount **/var/log** on **logs-lv** `sudo mount /dev/webdata-vg/logs-lv /var/log`


![image](images/Screenshot_17.png)

17. Run `sudo blkid` and copy the UUID with the **ext4** filesystem

![image](images/Screenshot_18.png)

18. Run `sudo vi /etc/fstab` to update the file so that the mount config will persist after restart of the server. Update the UUID that you copied.

![image](images/Screenshot_19.png)

19. Now, test the configuration by running `sudo mount -a` then reload `sudo systemctl daemon-reload`

![image](images/Screenshot_20.png)

## Installing Wordpres and configuring to use MySQL 

1. Install Wordpess on the Web Server EC2 instance

   - `sudo yum -y update` to update the repository
   - `sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json` to install wget, Apache and its dependencies.



![image](images/Screenshot_21.png)


![image](images/Screenshot_22.png)

  - `sudo systemctl enable httpd` then `sudo systemctl start httpd` to start Apache.

![image](images/Screenshot_23.png)

2. Install PHP and its dependencies
```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1

```

![image](images/Screenshot_24.png)

3. Configure SELinux Policies
```
 sudo chown -R apache:apache /var/www/html/wordpress
 sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
 sudo setsebool -P httpd_can_network_connect=1


```


![image](images/Screenshot_25.png)



4. Verify that the service is running `sudo systemctl status mysqld`



![image](images/Screenshot_27.png)

## Prepare the Database Server

**Launch a second RedHat instance, repeat the same steps as for the web server, use `db-lv` instead of `apps-lv`, and mount it to `/db` directory instead of `/var/www/html`**

1. Install MySQL on the newly created DB Server

   - `sudo systemctl retstart mysqld`

   - `sudo systemctl enable mysqld`

##

### Configure  DB to work with WorPress

```
sudo mysql
CREATE DATABASE wordpress;
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit

```
![image](images/Screenshot_28.png)

### Configure WordPress to connect to a remote database

  - Open MySQL on port **3306** on the DB Server and allow access to the DB Server only from your web server's private IP.

![image](images/Screenshot_29.png)

2. Install MySQL client and test that you can connect from the Web Server to the DB Server by using mysql-client.

  - `sudo yum install mysql`
  - `sudo mysql -u username -p -h <DB-Server-Private-IP-address>`



![image](images/Screenshot_31.png)

3. Change permissions and configuration so Apache can use WordPress

   - `cd /var/www/html`

   - `cd wordpress` then `ls ` 

   - `sudo vi wp-config.php`

   - Edit the following: `DB_NAME DB_USER DB_PASSWORD LOCALHOST`

   - enable port 80 from everywhere **0.0.0.0/0**
  
     

![image](images/Screenshot_33.png)

![image](images/Screenshot_35.png)
 
   

![image](images/Screenshot_32.png)

4. Access from your browser the link to your WordPress `http://<web-server0public-ip-addess>/wordpress/`

![image](images/Screenshot_37.png)

![image](images/Screenshot_38.png)


## THE END !!
