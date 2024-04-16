# Nextcloud

This is a tutorial about how i created a nextcloud "cloud" server on my home network for people to safely store their files and documents on my virtual machines.

# LINKS

I have followed the “centos 8 example install” from Nextcloud's official website:

https://docs.nextcloud.com/server/latest/admin_manual/installation/example_centos.html

https://docs.nextcloud.com/server/latest/admin_manual/maintenance/index.html

# Quick INFO About Components

`NEXTCLOUD:` File Sync and Share: Nextcloud is a popular open-source platform for file synchronization and sharing.

`Apache Server:` is a widely used and well-established web server. It's known for its reliability, security features, and flexibility.

`Maria DB:` It's commonly used for web applications due to its performance, stability, and ease of use.

`CLUSTER DB:` By replicating data from a master database to one or more slave databases, you ensure that if the master goes down, one of the slaves can take over, reducing downtime.

`BACKUP DB:` A backup server is crucial for data protection and recovery. Regular backups of Nextcloud data, configuration, and database ensure that in case of accidental deletion, corruption, or other issues, you can restore the system to a previous state. It's a key component for disaster recovery.

## VM STATUS

scripted-backup server/VM ip: 192.168.10.200

nextcloud server/VM ip: 192.168.10.103

master-database server/VM ip: 192.168.10.119

slave-database server/VM ip: 192.168.10.185

## INSTALLATION
### NEXTCLOUD
```
sudo dnf update -y
sudo dnf install -y epel-release yum-utils unzip curl wget \ bash-completion policycoreutils-python-utils mlocate bzip2
sudo dnf install -y httpd
sudo vi /etc/httpd/conf.d/nextcloud.conf
```
```
<VirtualHost *:80>
  DocumentRoot /var/www/html/nextcloud/
  ServerName  your.server.com

  <Directory /var/www/html/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews

    <IfModule mod_dav.c>
      Dav off
    </IfModule>

  </Directory>
</VirtualHost>
```
instead of your.server.com use your hostname (to find your hostname `sudo hostname`)
```
sudo systemctl enable httpd.service
sudo systemctl start httpd.service
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo dnf install yum-utils
sudo dnf module reset php
sudo dnf module install php:remi-8.2
sudo dnf update
sudo dnf install -y php php-cli php-gd php-mbstring php-intl php-pecl-apcu\ php-mysqlnd php-opcache php-json php-zip
sudo dnf install -y php-redis php-imagick #this command dosent work for me
sudo dnf install https://download.nextcloud.com/server/releases/latest.zip
sudo dnf install https://download.nextcloud.com/server/releases/latest.zip.md5
sudo md5sum  -c latest.zip.md5 < latest.zip
sudo unzip /root/latest.zip -d /var/www/html/
sudo cp -R nextcloud/ /var/www/html/ #this command dosent work for me
sudo mkdir /var/www/html/nextcloud/data
sudo chown -R apache:apache /var/www/html/nextcloud
sudo chmod -Rf 755 nextcloud
sudo chmod -Rf 777 nextcloud/sites/files
sudo chmod -R 755 /var/www/html/nextcloud
sudo chcon -R -t httpd_sys_content_rw_t /var/www/html/nextcloud
sudo chcon -R -t httpd_sys_content_t /var/www/html
sudo chcon -R -t httpd_sys_script_exec_t /usr/lib64/php/modules

sudo systemctl restart httpd.service
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --zone=public --add-service=https --permanent
sudo firewall-cmd --reload
sudo getenforce

sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/data(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/config(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/apps(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/.htaccess'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/.user.ini'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/3rdparty/aws/aws-sdk-php/src/data/logs(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/3rdparty(/.*)?'
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/var/www/html/nextcloud/lib(/.*)?'
sudo restorecon -R '/var/www/html/nextcloud/'
sudo setsebool -P httpd_can_network_connect on
sudo setsebool -P httpd_can_network_connect_db on
sudo setsebool -P httpd_can_sendmail on
sudo setsebool -P httpd_execmem on
sudo setsebool -P httpd_unified on
sudo setsebool -P httpd_read_user_content on
sudo setsebool -P httpd_enable_homedirs on
```

follow these steps: https://docs.nextcloud.com/server/latest/admin_manual/installation/installation_wizard.html
```
sudo vi /var/www/html/nextcloud/config/config.php
```
![image](https://github.com/Alex0424/Nextcloud/assets/33380655/edd21524-2b6c-4fcd-8d41-7329ca99b817)

### MARIADB
```
sudo dnf install -y mariadb mariadb-server
sudo systemctl enable mariadb.service
sudo systemctl start mariadb.service
sudo mysql_secure_installation
sudo mysql -u alex -p1234
MariaDB [(none)]> CREATE DATABASE nextcloud;
MariaDB [(none)]> CREATE USER  'alex'@'%' IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL ON nextcloud.* to 'alex'@'%' IDENTIFIED BY '1234' WITH GRANT OPTION;
MariaDB [(none)]> GRANT ALL ON *.* to 'alex'@'%' IDENTIFIED BY '1234' WITH GRANT OPTION;
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> EXIT;

extra:
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
sudo chown -R mysql:mysql /var/lib/mysql
sudo chmod -R 755 /var/lib/mysql
sudo systemctl restart mariadb.service
sudo systemctl status mariadb.services

sudo rsync -avP root@192.168.10.103:/var/www/html/nextcloud/ /home/alex/backup/nextcloud_$(date +%Y%m%d%H%M%S).bak

sudo rsync -Aax /home/alex/backup/nextcloud_20240206233215.bak/ root@192.168.10.103:/var/www/html/nextcloud
```

### BACKUP
Nextcloud server:
```
sudo vim backup_script.sh
#!/bin/bas
rsync -Aavx /var/www/html/nextcloud/ /home/alex/backup/nextcloud-dirbkp_`date +"%Y%m%d"`/
sudo chmod +x /home/alex/backup_script.sh
sudo crontab -e
30 * * * * bash /home/alex/backup_script.sh
```
Database server:
```
sudo vim backup_script_sql.sh
#!/bin/bas
mysqldump --single-transaction -h192.168.10.119 -u alex -p1234 nextcloud > /home/alex/nextcloud-sqlbkp_`date +"%Y%m%d"`.bak
sudo chmod +x /home/alex/backup_script_sql.sh
sudo crontab -e
30 * * * * bash /home/alex/backup_script_sql.sh
```
Backup server:
```
sudo dnf install mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
dnf install rsync
ssh-copy-id root@192.168.10.103
```
Backup script:
```
#!/bin/bas

#MARIADB NEXTCLOUD DATABASE BACKUP:
mysqldump --single-transaction -h192.168.10.119 -u alex -p1234 nextcloud > /home/alex/backup/nextcloud-sqlbkp_`date +"%Y%m%d"`.bak

#NEXTCLOUD BACKUP:
rsync -avP /var/www/html/nextcloud/ /home/alex/backup/nextcloud_$(date +%Y%m%d%H%M%S).bak
```
Restore script:
```
#!/bin/bash

#MARIADB RESTORE:
read -p "Entre full date of mysql backup folder in order to reset? " date

echo restoring nextcloud-sqlbkp_$date.bak backup folder...

mysql -h 192.168.10.119 -u alex -p1234 -e "DROP DATABASE nextcloud"

mysql -h 192.168.10.119 -u alex -p1234 -e "CREATE DATABASE nextcloud"

mysql -h 192.168.10.119 -u alex -p1234 nextcloud < nextcloud-sqlbkp_$date.bak

#NEXTCLOUD RESTORE:
read -p "Enter full date of nextcloud backup folder in order to reset? " date_2

echo restoring nextcloud-dirbkp_$date_2 backup folder...

rsync -Aax /home/alex/backup/nextcloud-dirbkp_$date_2/ /var/www/html/nextcloud/

echo bash script: done

sleep 5
```

### SLAVE REPLICATION
BOTH SERVER:
```
yum install mariadb-server mariadb
systemctl start mariadb
systemctl enable mariadb
mysql_secure_installation
```
MASTER:
```
firewall-cmd --add-port=3306/tcp --zone=public --permanent
firewall-cmd --reload
vim /etc/my.cnf
```
```
[mysqld]
server_id=1
log-basename=master
log-bin
binlog-format=row
binlog-do-db=nextcloud
[...]
```
```
systemctl restart mariadb
mysql -u root -p1234
CREATE DATABASE replica_db;
STOP SLAVE;
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY '1234';
FLUSH PRIVILEGES;
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;

mysqldump --all-databases --user=root --password --master-data > masterdatabase.sql
mysql -u root -p1234
UNLOCK TABLES;
quit
```
SLAVE:
```
scp masterdatabase.sql root@192.168.10.185:/home
vim /etc/my.cnf
```
```
[mysqld]
server-id = 2
replicate-do-db=nextcloud
[...]
```
```
mysql -u root -p1234 < /home/masterdatabase.sql
systemctl restart mariadb
mysql -u root -p1234
STOP SLAVE;
CHANGE MASTER TO MASTER_HOST='192.168.10.119', MASTER_USER='slave_user', MASTER_PASSWORD='1234', MASTER_LOG_FILE='mariadb-bin.000001', MASTER_LOG_POS=473;
START SLAVE;
SHOW SLAVE STATUS\G;
```

