# DEVOPS TOOLING WEBSITE SOLUTION

 ## PROJECT OVERVIEW

This project demonstrates the deployment of a 3-tier web application architecture using AWS. The architecture includes:

2 Web Servers (Apache + PHP) – for serving a tooling website

1 MySQL Database Server – for storing user and application data

1 NFS Server – to provide shared file storage across web servers

1 Apache Load Balancer – to distribute traffic evenly across both web servers
The objective is to build a scalable, fault-tolerant, and highly available infrastructure for hosting a PHP-based tooling application.

 # KEY TECHNOLOGIES & WHY THEY WERE USED

1. Amazon EC2

Used to create virtual servers to host each layer of the architecture. EC2 provides scalability and control over computing resources.

2. EBS Volumes

Used for persistent block storage. We attached EBS volumes to the NFS server so it can store shared web content and logs.

3. LVM (Logical Volume Manager)

LVM was used to group multiple EBS volumes into a single volume group and create flexible partitions. This makes it easier to manage, resize, and allocate storage dynamically.

4. XFS File System

We formatted the logical volumes with xfs, a high-performance file system optimized for large files and directories. It's the default file system in RHEL-based systems and ideal for a server environment.

5. NFS (Network File System)

Chosen to allow both web servers to access a centralized directory (/var/www and /var/log/httpd) hosted on the NFS server. This ensures consistency across all servers.

6. MySQL

MySQL was used to store backend data of the tooling website. It's lightweight, fast, and easy to manage.

7. Apache + PHP

Apache HTTP Server and PHP were installed on the web servers to serve dynamic content and connect to the MySQL backend.

8. Apache Load Balancer

Apache was configured on a separate EC2 instance to distribute web traffic between the two web servers using the lbmethod=byrequests policy.

## STEP ONE: SETTING UP THE NFS SERVER

1. **Launch an EC2 Instance**

   * Type: t3.micro
   * OS: RedHat 9
<img width="1920" height="827" alt="NFS server" src="https://github.com/user-attachments/assets/ee272209-f371-44e0-846d-6a19d06770af" />

2. **Attach 3 or 4 EBS Volumes** to the instance via AWS Console.
<img width="1920" height="827" alt="EBS Volumes Attached" src="https://github.com/user-attachments/assets/472b2800-2835-4944-b50b-da83b5570e82" />

3. **SSH into the Instance**

```bash
ssh -i <keypair.pem> ec2-user@<nfs-ip-address>
```

4. **Check Available Disks**

```bash
lsblk
```
<img width="1920" height="1080" alt="Screenshot (277)" src="https://github.com/user-attachments/assets/e850b39b-12cb-413f-ac03-8967aafcf8ad" />

5. **Partition Each Disk Using**

Repeat the following for each EBS disk:

```bash
sudo gdisk /dev/nvme1n1
# Press 'n' → ENTER through all prompts
# Then press 'w' to write and confirm
```
<img width="1920" height="1080" alt="Screenshot (278)" src="https://github.com/user-attachments/assets/dfe6cf50-3862-4fc3-8fc5-021499847054" />

6. **Install LVM2 and Create Physical Volumes**

```bash
sudo yum install lvm2 -y
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1 /dev/nvme4n1p1
```

7. **Create a Volume Group**

```bash
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1 /dev/nvme4n1p1
```
<img width="1920" height="1080" alt="Screenshot (280)" src="https://github.com/user-attachments/assets/0a0eef95-31f5-40ef-983d-200d99cd81f5" />

8. **Create Logical Volumes**
<img width="1920" height="1080" alt="Screenshot (281)" src="https://github.com/user-attachments/assets/7b2369d2-ef4e-4a26-892f-3cac83e5d3e0" />

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
<img width="1920" height="1080" alt="Screenshot (282)" src="https://github.com/user-attachments/assets/911d427c-da31-4235-bd0b-ef62a17286c3" />

10. **Create and Mount the Volumes**

```bash
sudo mkdir -p /mnt/apps /mnt/apt /mnt/logs
sudo mount /dev/webdata-vg/apps-lv /mnt/apps
sudo mount /dev/webdata-vg/apt-lv /mnt/apt
sudo mount /dev/webdata-vg/logs-lv /mnt/logs
```
<img width="1920" height="1080" alt="Screenshot (285)" src="https://github.com/user-attachments/assets/12a5062c-9fa4-4324-a862-317f28c7e29a" />

11. **Install and Start NFS Server**

```bash
sudo yum install nfs-utils -y
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
```
<img width="1920" height="1080" alt="Screenshot (287)" src="https://github.com/user-attachments/assets/ad8a3556-9f29-4a46-bf36-5fae7496efc6" />

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
<img width="1920" height="1080" alt="Screenshot (290)" src="https://github.com/user-attachments/assets/d90d680f-36a2-4e49-8224-d67d2ec5f71e" />


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
<img width="1920" height="1080" alt="Screenshot (292)" src="https://github.com/user-attachments/assets/c6839bf8-3a38-4275-b4ba-e8bcf9956b19" />

4. **Create Database and User**

```bash
sudo mysql
CREATE DATABASE tooling;
CREATE USER 'webaccess'@'172.31.0.0/16' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'172.31.0.0/16';
FLUSH PRIVILEGES;
exit;
```
<img width="1920" height="1080" alt="Screenshot (291)" src="https://github.com/user-attachments/assets/99432b25-3cfc-4ec6-b04d-28c939b66563" />

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
<img width="1920" height="1080" alt="Screenshot (293)" src="https://github.com/user-attachments/assets/7f07fed6-47e2-491a-a48d-69c1476d5dc3" />
<img width="1920" height="1080" alt="Screenshot (294)" src="https://github.com/user-attachments/assets/3d38b5b3-72d6-4bed-9f9b-a5f8260aa44f" />

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
<img width="1920" height="1080" alt="Screenshot (296)" src="https://github.com/user-attachments/assets/ad3d46c7-e07d-448f-82ab-31c1f0828d6c" />


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
<img width="1920" height="1080" alt="Screenshot (310)" src="https://github.com/user-attachments/assets/0136d699-7fb4-4fe2-81f9-7b39aa8b1918" />

2. **Configure MySQL Credentials** Update `functions.php` in `/var/www/html/includes/` to reflect MySQL server IP, user, and password.

3. **Import Database**

```bash
mysql -h <mysql-ip> -u webaccess -p tooling < tooling/tooling-db.sql
```
<img width="1920" height="1080" alt="Screenshot (301)" src="https://github.com/user-attachments/assets/29864da1-3ff2-4015-b9df-54509126955a" />

4. **Create an Admin User**

```bash
mysql -h <mysql-ip> -u Dimma -p
USE tooling;
INSERT INTO users (id, username, password, email, user_type, status) VALUES (1, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
exit;
```
<img width="1920" height="1080" alt="Screenshot (305)" src="https://github.com/user-attachments/assets/553acbf7-e104-4e0c-a7c6-31377d59c42d" />

5. **Restart Apache**

```bash
sudo systemctl restart httpd
```
<img width="1920" height="1080" alt="Screenshot (306)" src="https://github.com/user-attachments/assets/0a0bae61-4199-44c8-b73d-c5aedf9c0eac" />

6. **Test Web Application** Visit:

```
http://<webserver1-ip>/index.php
http://<webserver2-ip>/index.php
```

Login with:

* Username: `myuser`
* Password: `password`

✅ Web application is now deployed on both servers.
<img width="1920" height="1080" alt="Screenshot (307)" src="https://github.com/user-attachments/assets/8e72bcdc-aa8f-47a0-a852-26f4c275c7c6" />
<img width="1920" height="1080" alt="Screenshot (308)" src="https://github.com/user-attachments/assets/4c695fe0-c9cb-4668-9bd3-069a1285116e" />
<img width="1920" height="1080" alt="Screenshot (309)" src="https://github.com/user-attachments/assets/63c0a9e4-17ca-4443-8e8c-683f4b51eda1" />


## STEP FIVE: SETTING UP AN APACHE LOAD BALANCER

1. **Launch Ubuntu EC2 Instance for Load Balancer** Allow Port 80 in Security Group (0.0.0.0/0).

2. **Install Apache & Modules**

```bash
sudo apt update -y
sudo apt install apache2 libxml2-dev -y
sudo a2enmod rewrite proxy proxy_balancer proxy_http headers lbmethod_byrequests
sudo systemctl restart apache2
```
<img width="1920" height="1080" alt="Screenshot (311)" src="https://github.com/user-attachments/assets/6ce73e5d-5067-436e-b151-d977e236af85" />
<img width="1920" height="1080" alt="Screenshot (312)" src="https://github.com/user-attachments/assets/793f8d3f-a628-457e-98ad-c4829a0882f1" />

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

Update Apache config:

```
BalancerMember http://web1
BalancerMember http://web2
```
<img width="1920" height="1080" alt="Screenshot (313)" src="https://github.com/user-attachments/assets/338c3d51-12fe-4db6-88d0-2bcf0e89ce14" />

5. **Restart Apache and Test**

```bash
sudo systemctl restart apache2
curl http://web1
curl http://web2
```
<img width="1920" height="1080" alt="Screenshot (314)" src="https://github.com/user-attachments/assets/da5d8269-7d59-40dd-8c08-b2f2482109b9" />

6. **Check Logs to Verify Load Balancing**

```bash
sudo tail -f /var/log/httpd/access_log
```

✅ Load Balancer setup complete. Traffic now distributes between both servers.


🎉 **Project Completed: Fully functional 3-tier web application with NFS shared storage, centralized MySQL, and Apache Load Balancer.**
