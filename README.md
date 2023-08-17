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
  ### Step 1: Bash scripts
  Creat a new folder called Automated_provisionning
  Create a file Vagrantfile with the following: 
  ```
Vagrant.configure("2") do |config|
  config.hostmanager.enabled = true 
  config.hostmanager.manage_host = true
  
### DB vm  ####
  config.vm.define "db01" do |db01|
    db01.vm.box = "eurolinux-vagrant/centos-stream-9"
    db01.vm.hostname = "db01"
    db01.vm.network "private_network", ip: "192.168.56.15"
    db01.vm.provider "virtualbox" do |vb|
     vb.memory = "600"
   end
    db01.vm.provision "shell", path: "mysql.sh"  

  end
  
### Memcache vm  #### 
  config.vm.define "mc01" do |mc01|
    mc01.vm.box = "eurolinux-vagrant/centos-stream-9"
    mc01.vm.hostname = "mc01"
    mc01.vm.network "private_network", ip: "192.168.56.14"
    mc01.vm.provider "virtualbox" do |vb|
     vb.memory = "600"
   end
    mc01.vm.provision "shell", path: "memcache.sh"  
  end
  
### RabbitMQ vm  ####
  config.vm.define "rmq01" do |rmq01|
    rmq01.vm.box = "eurolinux-vagrant/centos-stream-9"
  rmq01.vm.hostname = "rmq01"
    rmq01.vm.network "private_network", ip: "192.168.56.16"
    rmq01.vm.provider "virtualbox" do |vb|
     vb.memory = "600"
   end
    rmq01.vm.provision "shell", path: "rabbitmq.sh"  
  end
  
### tomcat vm ###
   config.vm.define "app01" do |app01|
    app01.vm.box = "eurolinux-vagrant/centos-stream-9"
    app01.vm.hostname = "app01"
    app01.vm.network "private_network", ip: "192.168.56.12"
    app01.vm.provision "shell", path: "tomcat.sh"  
    app01.vm.provider "virtualbox" do |vb|
     vb.memory = "800"
   end
   end
   
  
### Nginx VM ###
  config.vm.define "web01" do |web01|
    web01.vm.box = "ubuntu/jammy64"
    web01.vm.hostname = "web01"
  web01.vm.network "private_network", ip: "192.168.56.11"
#  web01.vm.network "public_network"
  web01.vm.provider "virtualbox" do |vb|
     vb.gui = true
     vb.memory = "800"
   end
  web01.vm.provision "shell", path: "nginx.sh"  
end
  
end
```
We also need to have in the same folder the files: nginx.sh, rabitmq.sh, memcache.sh, mysql.sh and tomcat.sh with the following contents
- File 1: memcache.sh
  ```
  #!/bin/bash
  sudo dnf install epel-release -y
  sudo dnf install memcached -y
  sudo systemctl start memcached
  sudo systemctl enable memcached
  sudo systemctl status memcached
  sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
  sudo systemctl restart memcached
  firewall-cmd --add-port=11211/tcp
  firewall-cmd --runtime-to-permanent
  firewall-cmd --add-port=11111/udp
  firewall-cmd --runtime-to-permanent
  sudo memcached -p 11211 -U 11111 -u memcached -d
  ```
- File 2: tomcat.sh
  ```
  TOMURL="https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.75/bin/apache-tomcat-9.0.75.tar.gz"
  dnf -y install java-11-openjdk java-11-openjdk-devel
  dnf install git maven wget -y
  cd /tmp/
  wget $TOMURL -O tomcatbin.tar.gz
  EXTOUT=`tar xzvf tomcatbin.tar.gz`
  TOMDIR=`echo $EXTOUT | cut -d '/' -f1`
  useradd --shell /sbin/nologin tomcat
  rsync -avzh /tmp/$TOMDIR/ /usr/local/tomcat/
  chown -R tomcat.tomcat /usr/local/tomcat
  
  rm -rf /etc/systemd/system/tomcat.service
  
  cat <<EOT>> /etc/systemd/system/tomcat.service
  [Unit]
  Description=Tomcat
  After=network.target
  
  [Service]
  
  User=tomcat
  Group=tomcat
  
  WorkingDirectory=/usr/local/tomcat
  
  #Environment=JRE_HOME=/usr/lib/jvm/jre
  Environment=JAVA_HOME=/usr/lib/jvm/jre
  
  Environment=CATALINA_PID=/var/tomcat/%i/run/tomcat.pid
  Environment=CATALINA_HOME=/usr/local/tomcat
  Environment=CATALINE_BASE=/usr/local/tomcat
  
  ExecStart=/usr/local/tomcat/bin/catalina.sh run
  ExecStop=/usr/local/tomcat/bin/shutdown.sh
  
  
  RestartSec=10
  Restart=always
  
  [Install]
  WantedBy=multi-user.target
  
  EOT
  
  systemctl daemon-reload
  systemctl start tomcat
  systemctl enable tomcat
  
  git clone -b main https://github.com/devopshydclub/vprofile-project.git
  cd vprofile-project
  mvn install
  systemctl stop tomcat
  sleep 20
  rm -rf /usr/local/tomcat/webapps/ROOT*
  cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
  systemctl start tomcat
  sleep 20
  systemctl stop firewalld
  systemctl disable firewalld
  #cp /vagrant/application.properties /usr/local/tomcat/webapps/ROOT/WEB-INF/classes/application.properties
  systemctl restart tomcat
  ```
- File 3: rabbitmq.sh
  ```
  #!/bin/bash
  sudo yum install epel-release -y
  sudo yum update -y
  sudo yum install wget -y
  cd /tmp/
  dnf -y install centos-release-rabbitmq-38
   dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
   systemctl enable --now rabbitmq-server
   firewall-cmd --add-port=5672/tcp
   firewall-cmd --runtime-to-permanent
  sudo systemctl start rabbitmq-server
  sudo systemctl enable rabbitmq-server
  sudo systemctl status rabbitmq-server
  sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
  sudo rabbitmqctl add_user test test
  sudo rabbitmqctl set_user_tags test administrator
  sudo systemctl restart rabbitmq-server
  ```
- File 4: mysql.sh
  ```
  #!/bin/bash
  DATABASE_PASS='admin123'
  sudo yum update -y
  sudo yum install epel-release -y
  sudo yum install git zip unzip -y
  sudo yum install mariadb-server -y
  
  
  # starting & enabling mariadb-server
  sudo systemctl start mariadb
  sudo systemctl enable mariadb
  cd /tmp/
  git clone -b main https://github.com/devopshydclub/vprofile-project.git
  #restore the dump file for the application
  sudo mysqladmin -u root password "$DATABASE_PASS"
  sudo mysql -u root -p"$DATABASE_PASS" -e "UPDATE mysql.user SET Password=PASSWORD('$DATABASE_PASS') WHERE User='root'"
  sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
  sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
  sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
  sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
  sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
  sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
  sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
  sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-project/src/main/resources/db_backup.sql
  sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
  
  # Restart mariadb-server
  sudo systemctl restart mariadb
  
  
  #starting the firewall and allowing the mariadb to access from port no. 3306
  sudo systemctl start firewalld
  sudo systemctl enable firewalld
  sudo firewall-cmd --get-active-zones
  sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
  sudo firewall-cmd --reload
  sudo systemctl restart mariadb
  ```
- File 5: nginx.sh
  ```
  # adding repository and installing nginx		
  apt update
  apt install nginx -y
  cat <<EOT > vproapp
  upstream vproapp {
  
   server app01:8080;
  
  }
  
  server {
  
    listen 80;
  
  location / {
  
    proxy_pass http://vproapp;
  
  }
  
  }
  
  EOT
  
  mv vproapp /etc/nginx/sites-available/vproapp
  rm -rf /etc/nginx/sites-enabled/default
  ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
  
  #starting nginx service and firewall
  systemctl start nginx
  systemctl enable nginx
  systemctl restart nginx
  ```
  
  

## Technologies 
- Spring MVC
- Spring Security
- Spring Data JPA
- Maven
- JSP
- MySQL

# Results
![Login](https://github.com/Ndzenyuy/project-1_2-Local-deployment_of_web_app/blob/main/images/Screenshot%20from%202023-07-10%2016-23-33.png)  
![Home page](https://github.com/Ndzenyuy/project-1_2-Local-deployment_of_web_app/blob/main/images/Screenshot%20from%202023-07-10%2016-25-20.png)
![](https://github.com/Ndzenyuy/project-1_2-Local-deployment_of_web_app/blob/main/images/Screenshot%20from%202023-07-10%2016-26-26.png)
![](https://github.com/Ndzenyuy/project-1_2-Local-deployment_of_web_app/blob/main/images/Screenshot%20from%202023-07-10%2016-27-03.png)
![](https://github.com/Ndzenyuy/project-1_2-Local-deployment_of_web_app/blob/main/images/Screenshot%20from%202023-07-10%2016-43-00.png)




