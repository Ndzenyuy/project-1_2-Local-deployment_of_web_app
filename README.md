# Project-#1 & #2: Manual and automated provisionning of a Web App on a local machine
  This project sets up a web app in on a local machine using vagrant to configure the differnt servers on Virtualbox, these VMs interrract with each other via HTTP protocole. 
  Users login to the site, their credentials are stored in the MySql database, each time data is fetched from the data base, a copy is cached in memcached offload the database from being overwhelmed with many reads, thus optimising data reads. RabbitMq is used to couple between the services.
  ## Services involved: 
  - Tomcat
  - Nginix
  - RabbitMQ
  - Memcached
  - MySql
    
# Project Architecture
![project architecture](https://github.com/Ndzenyuy/project-1_2-Local-deployment_of_web_app/blob/main/images/Screenshot%20from%202023-07-12%2012-08-01.png)

  ## Manual provisionning
  We use vagrant to provision the VMs, then on ssh into each of them, install all the required software, updates and configurations
  ### Prerequisites
    - Oracle VM Virtualbox
    - Vagrant
    - Vagrant plugins (vagrant-hostmanager)
    - Git bash or equivalent editor
    
  ### Step 1: Clone source code
   Run the following command to clone the project Repo: 
   ```
      git clone -b main --single-branch https://github.com/Ndzenyuy/vprofile-project
   ```
  Then cd into vagrant/Manual_provisioning
  ### Step 2: Bring up VMs
  ```
  vagrant up
  ```
  ### Provisioning
  
  1. MySQL(Database Service) setup
     Login to the db vm
     ```
     vagrant ssh db01
     ```
     Verify Hosts entry, if entries missing update the it with IP and hostnames
     
    cat /etc/hosts
  Update OS with latest patches
  ```
    yum update -y
  ```
  Set Repository
  ```
    yum install epel-release -y
  ```
  Install Maria DB Package
    ```
    yum install git mariadb-server -y
    ```
  Starting & enabling mariadb-server
    ```
    systemctl start mariadb
    systemctl enable mariadb
      ```
NOTE: Set db root password, I will be using _admin123_ as password

Set DB name and users.
```
# mysql -u root -padmin123
mysql> create database accounts;
mysql> grant all privileges on accounts.* TO 'admin'@’%’ identified by 'admin123' ;
mysql> FLUSH PRIVILEGES;
mysql> exit;
```
Download Source code & Initialize Database.
```
# git clone -b main https://github.com/hkhcoder/vprofile-project.git
# cd vprofile-project
# mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
# mysql -u root -padmin123 accounts
mysql> show tables;
```
Restart mariadb-server
```
# systemctl restart mariadb
```

Starting the firewall and allowing the mariadb to access from port no. 3306
```
# systemctl start firewalld
# systemctl enable firewalld
# firewall-cmd --get-active-zones
# firewall-cmd --zone=public --add-port=3306/tcp --permanent
# firewall-cmd --reload
# systemctl restart mariadb
```
  2. Memcached(DB caching service)
     Install, start & enable memcache on port 11211
```
# sudo dnf install epel-release -y
# sudo dnf install memcached -y
# sudo systemctl start memcached
# sudo systemctl enable memcached
# sudo systemctl status memcached
# sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
# sudo systemctl restart memcached
```
Starting the firewall and allowing the port 11211 to access memcache
```
# firewall-cmd --add-port=11211/tcp
# firewall-cmd --runtime-to-permanent
# firewall-cmd --add-port=11111/udp
# firewall-cmd --runtime-to-permanent
# sudo memcached -p 11211 -U 11111 -u memcached -d
```
  3. RabbitMQ (Broker/Queue service)
     Login to the RabbitMQ vm
```
$ vagrant ssh rmq01
```
Verify Hosts entry, if entries missing update the it with IP and hostnames
```
# cat /etc/hosts
```
Update OS with latest patches
```
# yum update -y
```
Set EPEL Repository
```
# yum install epel-release -y
```
Install Dependencies
```
# sudo yum install wget -y
# cd /tmp/
# dnf -y install centos-release-rabbitmq-38
# dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
# systemctl enable --now rabbitmq-server
# firewall-cmd --add-port=5672/tcp
# firewall-cmd --runtime-to-permanent
# sudo systemctl start rabbitmq-server
# sudo systemctl enable rabbitmq-server
# sudo systemctl status rabbitmq-server
# sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
# sudo rabbitmqctl add_user test test
# sudo rabbitmqctl set_user_tags test administrator
# sudo systemctl restart rabbitmq-server
```
  4. Tomcat (Application Service)
     
Login to the tomcat vm
```
$ vagrant ssh app01
```
Verify Hosts entry, if entries missing update the it with IP and hostnames
```
# cat /etc/hosts
```
Update OS with latest patches
```
# yum update -y
```
Set Repository
```
# yum install epel-release -y
```
Install Dependencies
```
# dnf -y install java-11-openjdk java-11-openjdk-devel
# dnf install git maven wget -y
```
Change dir to /tmp
```
# cd /tmp/
```
Download & Tomcat Package
```
# wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.75/bin/apache-tomcat-9.0.75.tar.gz
# tar xzvf apache-tomcat-9.0.75.tar.gz
```
Add tomcat user
```
# useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
```
Copy data to tomcat home dir
```
# cp -r /tmp/apache-tomcat-9.0.75/* /usr/local/tomcat/
```
Make tomcat user owner of tomcat home dir
```
# chown -R tomcat.tomcat /usr/local/tomcat# Setup systemctl command for tomcat
```
Update file with following content.
```
# vi /etc/systemd/system/tomcat.service
```
    [Unit]
    Description=Tomcat
    After=network.target
    [Service]
    User=tomcat
    WorkingDirectory=/usr/local/tomcat
    Environment=JRE_HOME=/usr/lib/jvm/jre
    Environment=JAVA_HOME=/usr/lib/jvm/jre
    Environment=CATALINA_HOME=/usr/local/tomcat
    Environment=CATALINE_BASE=/usr/local/tomcat
    ExecStart=/usr/local/tomcat/bin/catalina.sh run
    ExecStop=/usr/local/tomcat/bin/shutdown.sh
    SyslogIdentifier=tomcat-%i
    [Install]
    WantedBy=multi-user.target
Reload systemd files
```
    # systemctl daemon-reload
```
Start & Enable service
```
    # systemctl start tomcat
    # systemctl enable tomcat
```
Enabling the firewall and allowing port 8080 to access the tomcat
```
# systemctl start firewalld
# systemctl enable firewalld
# firewall-cmd --get-active-zones
# firewall-cmd --zone=public --add-port=8080/tcp --permanent
# firewall-cmd --reload
```
  4. Nginx (Web Service)
     
Login to the Nginx vm
```
$ vagrant ssh web01
```
Verify Hosts entry, if entries missing update the it with IP and hostnames
```
# cat /etc/hosts
```
Update OS with latest patches
```
# apt update
# apt upgrade
```
Install nginx
```
# apt install nginx -y
```
Create Nginx conf file with below content
```
# vi /etc/nginx/sites-available/vproapp
```
    upstream vproapp {
    server app01:8080;
    }
    server {
    listen 80;
    location / {
    proxy_pass http://vproapp;
    }
    }
    
Remove default nginx conf
```
# rm -rf /etc/nginx/sites-enabled/default
```
Create link to activate website
```
# ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
```
Restart Nginx
```
# systemctl restart nginx
```

  ## Automated provisionning
  Using bash scripting, the setup was configured for automatic configuration and installations
  

## Technologies 
- Spring MVC
- Spring Security
- Spring Data JPA
- Maven
- JSP
- MySQL

# Results
![Login](https://github.com/Ndzenyuy/vprofile-project/blob/main/images/Screenshot%20from%202023-07-10%2016-23-33.png)  
![Home page](https://github.com/Ndzenyuy/vprofile-project/blob/main/images/Screenshot%20from%202023-07-10%2016-25-20.png)




