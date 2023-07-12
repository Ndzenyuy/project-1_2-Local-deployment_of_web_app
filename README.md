# Project-#1 & #2: vprofile manual and automated provisionning on local machine
  This project sets up a web app in on a local machine using vagrant to configure the differnt servers on Virtualbox, these VMs interrract with each other via HTTP protocole. User login to the site, their credentials are stored in the MySql database, the each time any data is sought from the data base, a copy is cached in memcached offload the database from being overwhelmed with many reads, thus optimising data reads. RabbitMq is used to couple between the services.
  ## Services involved: 
  - Tomcat
  - Nginix
  - RabbitMQ
  - Memcached
  - MySql
  ## Manual provisionning
  We use vagrant to provision the VMs, then on ssh into each of them, install all the required software, updates and configurations
  ## Automated provisionning
  Using bash scripting, the setup was configured for automatic configuration and installations
# Project Architecture
![project architecture](https://github.com/Ndzenyuy/vprofile-project/blob/main/images/Screenshot%20from%202023-07-12%2012-08-01.png)
  

## Technologies 
- Spring MVC
- Spring Security
- Spring Data JPA
- Maven
- JSP
- MySQL

# Results
![Login](https://github.com/Ndzenyuy/vprofile-project/blob/main/images/Screenshot%20from%202023-07-10%2016-23-33.png)   ![Home page](https://github.com/Ndzenyuy/vprofile-project/blob/main/images/Screenshot%20from%202023-07-10%2016-25-20.png)




