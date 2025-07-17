# DEVOPS TOOLING WEBSITE SOLUTION

## TASK

To implement a 2-tier web application architecture with a single database (MySQL) and an NFS server as shared file storage.


## STEP ONE: SETTING UP THE NFS SERVER

1. **Launch an EC2 Instance**

   * Type: t3.micro
   * OS: RedHat 9

2. **Attach 3 or 4 EBS Volumes** to the instance via AWS Console.

3. **SSH into the Instance**

```bash
ssh -i <keypair.pem> ec2-user@<nfs-ip-address>
```

4. **Check Available Disks**

```bash
lsblk
```

5. **Partition Each Disk Using**

Repeat the following for each EBS disk:

```bash
sudo gdisk /dev/nvme1n1
# Press 'n' → ENTER through all prompts
# Then press 'w' to write and confirm
```

6. **Install LVM2 and Create Physical Volumes**

```bash
sudo yum install lvm2 -y
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1 /dev/nvme4n1p1
```

7. **Create a Volume Group**

```bash
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1 /dev/nvme4n1p1
```

8. **Create Logical Volumes**

```bash
sudo lvcreate -n apps-lv -L 9G webdata-vg
sudo lvcreate -n apt-lv -L 9G webdata-vg
sudo lvcreate -n logs-lv -L 9G webdata-vg
```

9. **Format Logical Volumes with XFS**

```bash
sudo mkfs.xfs /dev/webdata-vg/apps-lv
sudo mkfs.xfs /dev/webdata-vg/apt-lv
sudo mkfs.xfs /dev/webdata-vg/logs-lv
```

10. **Create and Mount the Volumes**

```bash
sudo mkdir -p /mnt/apps /mnt/apt /mnt/logs
sudo mount /dev/webdata-vg/apps-lv /mnt/apps
sudo mount /dev/webdata-vg/apt-lv /mnt/apt
sudo mount /dev/webdata-vg/logs-lv /mnt/logs
```

11. **Install and Start NFS Server**

```bash
sudo yum install nfs-utils -y
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
```

12. **Set Ownership and Permissions**

```bash
sudo chown -R nobody:nobody /mnt/apps /mnt/apt /mnt/logs
sudo chmod -R 777 /mnt/apps /mnt/apt /mnt/logs
```

13. **Configure NFS Exports**

```bash
sudo vi /etc/exports
```

Add the following:

```
/mnt/apps 172.31.0.0/16(rw,sync,no_root_squash,no_subtree_check)
/mnt/apt 172.31.0.0/16(rw,sync,no_root_squash,no_subtree_check)
/mnt/logs 172.31.0.0/16(rw,sync,no_root_squash,no_subtree_check)
```

Then run:

```bash
sudo exportfs -ra
sudo systemctl restart nfs-server
```


## STEP TWO: SETTING UP THE MYSQL DATABASE SERVER

1. **Launch Ubuntu EC2 Instance**

2. **Install MySQL Server**

```bash
sudo apt update -y
sudo apt install mysql-server -y
```

3. **Configure Remote Access**

```bash
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```

Change:

```
bind-address = 0.0.0.0
```

4. **Create Database and User**

```bash
sudo mysql
CREATE DATABASE tooling;
CREATE USER 'Dimma'@'172.31.0.0/16' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON tooling.* TO 'Dimma'@'172.31.0.0/16';
FLUSH PRIVILEGES;
exit;
```

5. **Allow Port 3306 in Security Group** Allow traffic from webservers’ subnet.


## STEP THREE: CONFIGURING THE WEB SERVERS

1. **Launch Two RedHat EC2 Instances**

2. **Install Required Packages**

```bash
sudo yum install nfs-utils nfs4-acl-tools httpd -y
```

3. **Mount NFS Shares**

```bash
sudo mkdir -p /var/www /var/log/httpd
sudo mount -t nfs <nfs-ip>:/mnt/apps /var/www
sudo mount -t nfs <nfs-ip>:/mnt/logs /var/log/httpd
```

4. **Make Mounts Persistent** Edit fstab:

```bash
sudo vi /etc/fstab
```

Add:

```
<nfs-ip>:/mnt/apps /var/www nfs defaults 0 0
<nfs-ip>:/mnt/logs /var/log/httpd nfs defaults 0 0
```

5. **Unmount Shared Logs and Recreate Local Log Directory**

Unmounting the NFS log directory allows Apache to regain write permissions to the local `/var/log/httpd` directory for proper logging:

```bash
sudo umount /var/log/httpd
sudo mkdir -p /var/log/httpd
sudo chown apache:apache /var/log/httpd
```

6. **Install Apache and PHP with REMI Repository**

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm
sudo dnf module reset php -y
sudo dnf module enable php:remi-7.4 -y
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd -y
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```

7. **Set SELinux Policies**

```bash
sudo setsebool -P httpd_use_nfs 1
sudo setsebool -P httpd_can_network_connect 1
```

8. **Verify Files Appear Across Servers** Create a test file on one server and check if it appears on the other.

```bash
sudo touch /var/www/testfile.txt
```

## STEP FOUR: DEPLOY THE TOOLING WEBSITE

1. **Install Git and Clone Repository**

```bash
sudo yum install git -y
git clone https://github.com/citadelict/tooling.git
sudo cp -r tooling/html/* /var/www/html/
```

2. **Configure MySQL Credentials** Update `functions.php` in `/var/www/html/includes/` to reflect MySQL server IP, user, and password.

3. **Import Database**

```bash
mysql -h <mysql-ip> -u Dimma -p tooling < tooling/tooling-db.sql
```

4. **Create an Admin User**

```bash
mysql -h <mysql-ip> -u Dimma -p
USE tooling;
INSERT INTO users (id, username, password, email, user_type, status) VALUES (1, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
exit;
```

5. **Restart Apache**

```bash
sudo systemctl restart httpd
```

6. **Test Web Application** Visit:

```
http://<webserver1-ip>/index.php
http://<webserver2-ip>/index.php
```

Login with:

* Username: `myuser`
* Password: `password`

✅ Web application is now deployed on both servers.


## STEP FIVE: SETTING UP AN APACHE LOAD BALANCER

1. **Launch Ubuntu EC2 Instance for Load Balancer** Allow Port 80 in Security Group (0.0.0.0/0).

2. **Install Apache & Modules**

```bash
sudo apt update -y
sudo apt install apache2 libxml2-dev -y
sudo a2enmod rewrite proxy proxy_balancer proxy_http headers lbmethod_byrequests
sudo systemctl restart apache2
```

3. **Configure Load Balancer Virtual Host**

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```

Add:

```apache
<Proxy "balancer://mycluster/">
  BalancerMember http://web1
  BalancerMember http://web2
  ProxySet lbmethod=byrequests
</Proxy>
ProxyPass "/" "balancer://mycluster/"
ProxyPassReverse "/" "balancer://mycluster/"
```

4. \*\*Set Local DNS in \*\***`/etc/hosts`**

```bash
sudo vi /etc/hosts
```

Add:

```
<web1-private-ip> web1
<web2-private-ip> web2
```

5. **Restart Apache and Test**

```bash
sudo systemctl restart apache2
curl http://web1
curl http://web2
```

6. **Check Logs to Verify Load Balancing**

```bash
sudo tail -f /var/log/httpd/access_log
```

✅ Load Balancer setup complete. Traffic now distributes between both servers.


🎉 **Project Completed: Fully functional 3-tier web application with NFS shared storage, centralized MySQL, and Apache Load Balancer.**
