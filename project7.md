
### DevOps Tooling Website Solution

The website will display a dashboard of some of the most useful DevOps tools as listed below:
- Jenkins
- Kubernetes
- JFrog Artifactory
- Rancher
- Docker
- Grafana
- Prometheus
- Kibana

The project will use the following components:
- AWS as infrastructure
- Web Server with Red Hat Enterprise Linux 8
- DB Server with Ubuntu 20.04 and MySQL
- Storage solution: Red Hat Enterprise Linux 8 + NFS Server


#### Step 1: Create NFS Server

##### Create an EC2 instance with RHEL Linux 8 Operating System

##### Configure LVM on the server: 
- Create three logical volumes of 'xfs' format and name them as lv-opt, lv-apps and lv-logs.
- Create three mount points for the logical volumes on the /mnt directory: lv-apps on /mnt/apps for webservers; lv-logs on /mnt/logs for webserver logs and lv-opt on /mnt/opt for jenkins server to be used in the next project



![Screen Shot 2021-05-21 at 9 32 32 AM](https://user-images.githubusercontent.com/44268796/119149107-2f935480-ba1b-11eb-8cbc-369bf5dbf387.png)


![Screen Shot 2021-05-21 at 9 42 52 AM](https://user-images.githubusercontent.com/44268796/119149111-315d1800-ba1b-11eb-9593-152b133a674c.png)


![Screen Shot 2021-05-21 at 9 56 48 AM](https://user-images.githubusercontent.com/44268796/119149115-328e4500-ba1b-11eb-8691-c8e0c0b3af49.png)


![Screen Shot 2021-05-21 at 10 25 53 AM](https://user-images.githubusercontent.com/44268796/119153105-f0ff9900-ba1e-11eb-9157-ad8a510ce83a.png)



##### Next step is to install the NFS server on the EC2 instance and activate it
```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

![Screen Shot 2021-05-21 at 10 04 23 AM](https://user-images.githubusercontent.com/44268796/119149917-f4455580-ba1b-11eb-8b53-b8e8f1b89e23.png)


##### Change permissions on NFS server to allow web servers to read, write and execute:
```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

##### Find the subnet CIDR of the NFS server and allow access to clients within the same subnet:
```
sudo vi /etc/exports

/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

sudo exportfs -arv
```

![Screen Shot 2021-05-21 at 11 22 37 AM](https://user-images.githubusercontent.com/44268796/119161029-dc270380-ba26-11eb-9e25-43e13337bba9.png)

```
sudo systemctl restart nfs-server.service
```
##### Check the port currently used by NFS and also edit the security group rules to allow inbound traffic on TCP 111, UDP 111, UDP 2049
```
rpcinfo -p | grep nfs
```


![Screen Shot 2021-05-21 at 11 25 38 AM](https://user-images.githubusercontent.com/44268796/119161635-8acb4400-ba27-11eb-9064-420bb62aab99.png)


#### Step 2: Creating and configuring the Database Server

##### Create an EC2 instance of type Ubuntu 20.04 and install MySQL

Install MySQL server:
```
sudo apt install mysql-server -y
```
```
sudo mysql_secure_installation
```

Log into MySQL and create a database called 'tooling'

![Screen Shot 2021-05-21 at 1 32 11 PM](https://user-images.githubusercontent.com/44268796/119176470-2618e500-ba39-11eb-9f24-61b23e81b053.png)


Next, create a database user called webaccess and grant permissions
Open the /etc/mysql/mysql.conf.d/mysqld.cnf file to edit the bind address to grant access to the 'webaccess' user from the webservers.

#### Step 3: Create and configure three identical webservers

##### Create three EC2 instances with Red Hat Linux 8 operating system

- Install NFS client 
```
sudo yum install nfs-utils nfs4-acl-tools -y
```
- Create a directory /var/www and mount it to target the NFS server’s export for apps
```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
```

![Screen Shot 2021-05-21 at 3 17 38 PM](https://user-images.githubusercontent.com/44268796/119187864-e3aad480-ba47-11eb-9cda-225fb86d71ac.png)


Add the following entry to /etc/fstab file so the changes persist:
```
<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
```
Install Apache:
```
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

Check the /var/html folder to verify that the Apache files and directories are present. Check the /mnt/apps directory on NFS server to verify if the same Apache files and directories are present.

- Mount the log folder for Apache on webserver to NFS folder for /mnt/logs with same steps as above. Update the /etc/fstab and make sure changes persist.

![Screen Shot 2021-05-21 at 3 45 27 PM](https://user-images.githubusercontent.com/44268796/119190816-d7287b00-ba4b-11eb-8787-568e0f137db4.png)


- Install git and clone the tooling source code at https://github.com/darey-io/tooling.git
``` 
sudo yum install git -y
git clone https://github.com/darey-io/tooling.git
```
###### After deploying the tooling repository code to the webserver, the html code is deployed to /var/www/html 


![Screen Shot 2021-05-22 at 9 53 40 PM](https://user-images.githubusercontent.com/44268796/119260324-50d17d80-bba0-11eb-9b26-e3cbe1f911e9.png)


###### PHP and its dependencies are installed

![Screen Shot 2021-05-22 at 9 56 23 PM](https://user-images.githubusercontent.com/44268796/119260434-bd4c7c80-bba0-11eb-9fcf-f506e8fa0fa5.png)



![Screen Shot 2021-05-22 at 9 57 32 PM](https://user-images.githubusercontent.com/44268796/119260435-bf164000-bba0-11eb-8f00-f2d30624a70e.png)



###### On the DB server, a user is created with credentials.

![Screen Shot 2021-05-22 at 9 59 28 PM](https://user-images.githubusercontent.com/44268796/119260438-c0e00380-bba0-11eb-96e3-a341ff24d225.png)


###### On the browser, the tooling website is accessible at this point




![Screen Shot 2021-05-22 at 10 00 10 PM](https://user-images.githubusercontent.com/44268796/119260518-1ddbb980-bba1-11eb-847a-5b626e843a46.png)




![Screen Shot 2021-05-22 at 10 00 22 PM](https://user-images.githubusercontent.com/44268796/119260522-203e1380-bba1-11eb-97fd-50a446f26d42.png)
































