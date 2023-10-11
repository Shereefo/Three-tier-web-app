![How_to_build_a_highly_available_web_application_with_AWS🌐-2](https://github.com/Shereefo/AWS-Projects/assets/137960467/4a88f634-d48a-4bf5-a7f7-26e0fc7b7c33)
# Introduction 
Scalability and high availability are critical core aspects of developing strong cloud infrastructure. No matter the web app, scalability is paramount to withstanding the continuously increasing number of users accessing it. High availability for better performance and data resiliency in case of any failures within an availability zone (AZ). Therefore to be highly available and scalable the application must be able to operate in multiple AZs with the proper operationally efficient AWS resources. In this project the goal is to deploy a web application which will be the very popular, Wordpress, a free opensource software in which you can install your own webhost.
# About The Application 
The goal is to deploy Wordpress's underlying AWS components independently of each other so they can scale without any interdependencies. This commonly reffered to as microservices architechture, allowing you to allocate more resources to parts of the application experiencing high demand without affecting other components. Also if one component fails or experiences issues it does not affect the entire application. Hence we will deploy a file server, application server, and the database server in this independent format. The application's components will be deployed in two AZs for high avaialabilty and fault tolerance. It is also essential to ensure the application is deployed in a stateless manner so that the components do not rely on or store session specific data or store information locally, and instead the data is shared across components in a way that any microservice can handle any request. 
# Steps In Summation 
1. Create a VPC (Virtual Private Network) across two availability zones (AZs) in a region
2. Deploy a highly available relational database across the AZs using RDS (Relational Database Service)
3. Create a caching layer using Elasticache, which will provide support and increasing performance for your database by functioning as its cache around it for frequent queries speeding up HTTP response time. This will reduce the pressure/load on the database. This is the completion of the data tier.
4. For the application tier, you will use the the NFS protocol to process a shared storage solution layer. With EFS (Elastic File System) you will create an NFS cluster across the AZs.
5. To complete the application tier create a load balanced group of web servers using an application load balancer and an EC2 launch template, that will auto scale in response to load.
# Create a VPC
The main part of the VPC is its underlying subnets as that is where you deploy your AWS resources, I configured three in each AZ totalling to six across the whole VPC. 

Each AZ contains:

**One public subnet**: Application load balancer (ALB) and NAT gateway
The ALB is in a public subnet as it is designed to distribute the traffic to the target group. As for the NAT gateway, it is there for enabaling outbound internet access for the resources in the private subnet without directly exposing them to it.

**Two private subnets**: Application and Data tiers are placed in a private subnet as this minimizes the exposure of the resources running your application to potential threats and unauthorized access from the internet. 

# <img width="1464" alt=" " src="https://github.com/Shereefo/AWS-Projects/assets/137960467/a6b057e0-df40-406b-931e-c3a38a4a3e15">

# ![image](https://github.com/Shereefo/AWS-Projects/assets/137960467/6ba5f200-df32-45a4-9c01-3d37a06c3ec4)


After configuring the subnets I attatched one internet gateway to the VPC for transporting public traffic. And deployed two NAT gateways in both public subnets. The NAT gateways are placed there as they translate the private IP addresses of resources in the private subnets into public IPs when they want to access the internet. Below shows one of the private subnets routing to the NAT Gateway. 

<img width="1031" alt="Screenshot 2023-08-31 at 3 32 30 PM" src="https://github.com/Shereefo/AWS-Projects/assets/137960467/ea0a3df2-1ac4-43d3-8bd5-350676441e25">

Without correctly configured route tables the subnets cannot associate themselves with a gateway rendering them unable to connect. Therefore I configured six different route tables and then edited their routes in the VPC dashboard. I specified the target in each route table to direct the subnets to either an internet or NAT gateway. 


<img width="1264" alt="Route Tables " src="https://github.com/Shereefo/AWS-Projects/assets/137960467/a312569d-78b9-483d-9ece-73dbe6ad5540">

# Configuring The Database
Before creating the database go to VPC and create two **security groups** for it named:
1. Wordpress Database SG
2. Wordpress Client SG 

You must edit the **inbound rule** of Wordpress Database SG of rule type **MySQL/Aurora** - allowing traffic on **port 3306** from WP Database Client SG. (Do not configure any inbound rules for the WP database Client SG) 

**RDS** is the database to deploy as it is highly available and fully managed. These benefits can greatly simplify deployment of the application's data layer while being available and reilable. Since we want to utilize RDS's highly available functionality we deploy two RDS instances in two different AZs. 

When configuring the database deployment, specify a subnet group. This tells RDS which subnets it can deploy its instances. You must add the two previously created private data subnets to the subnet group, this requires you to copy the subnet ID to then add them to the new subnet group. 
<img width="1179" alt="Subnet group" src="https://github.com/Shereefo/AWS-Projects/assets/137960467/c9d1153e-56cb-4368-add2-3104fbd0fdd7">

After forming the subnet group you can now launch RDS.
When creating the database:
1. choose **Aurora** as the RDS engine and then specify the **MySQL-compatible engine**.
2. choose the production template
3. create a DB password using auto generate or your own
4. configure the instance size (memory optimized db.r5.large is suitable)
5. configure **multi-AZ** this (neccessary to deploy read replicas for high availability)
6. select the VPC and database subnet group that was previously created
7. choose the **security group** WP Database SG
8. under additional configuration name the initial database name to wordpress
9. create the database

Configured Database cluster: 

<img width="1118" alt="Aurora cluster " src="https://github.com/Shereefo/AWS-Projects/assets/137960467/f3052132-4b56-4cde-b84b-10ef91880045">



# Add a Caching Layer
Since the database has to hold a lot of information regarding users and configurations it can undergo many non essential queries that induce a lot of strain on the databse. Therefore introducing a caching layer, (for this project **Elasticache Memcached**) would serve as a good solution to mitigate that, as it can cache frequently accessed data in memory that your application relies on. This allows for quicker retrieval than database querying.

Just like the database, before creating Elasticache we must create two security groups for its instances named: 
1. WP Cache SG
2. WP Cache Client SG

You must edit the **inbound rule** of WP Cache SG with the rule type **Custom TCP Rule** allowing traffic on **port 11211** from the Cache Client SG

Now to create the Elasticache instance: 
1. Go to Elasticache console navigate to Memcached clusters
2. Create a Memcached cluster
3. Name the cluster Wordpress-Memcached or something similar
4. leave the cluster settings default
5. create a new subnet group under subnet group settings and chose the data subnets from both AZs
6. under advanced settings select the securtiy group WP Cache SG

Configured caching layer: 

<img width="1142" alt="Elasticache" src="https://github.com/Shereefo/AWS-Projects/assets/137960467/007e0677-25aa-44aa-a624-9d0b4ac2206e">



# Configure the Shared Filesystem
Using Elastic Filesystem (EFS) you can deploy a cluster that will provide a shared filesystem for your web application's servers. A shared file system can facilitate data sharing and communication between services by providing a common data store. It is also great for scaling, when the web application (Wordpress) sccales horizontally by adding more servers, a shared file system simplifies data sharing among the new instances, ensuring all instances have access to the same data resources.  

Again we must configure two security groups before deploying a cluster. This will be named like the database and cache cluster: 
1. WP FS SG
2. WP FS Client SG

You must edit the **inbound rule** of WP FS SG with the rule type **NFS** allowing traffic on **port 2049** from the WP FS Client SG 

Now to create the EFS cluster: 
1. go to the EFS console and select create file system
2. select your VPC and hit customize
3. in the network access step, under mount targets chose application subnet A and B (one for AZ A and the other for AZ B) Associate the security group WP FS SG with each mount target.
4. accpet the defualts for file system policy, then select create to configure the two mount targets in the subnets

Configured file system:

<img width="1184" alt="EFS" src="https://github.com/Shereefo/AWS-Projects/assets/137960467/9fadc51b-234f-45d8-9683-2a8790eb6cc1">
<img width="1144" alt="Mount Target" src="https://github.com/Shereefo/AWS-Projects/assets/137960467/405790ec-478f-47e1-9720-f75445d23430">

# Deploy the Application 
To deploy the application itself in a highly available manner, create an EC2 auto scaling group which will add and remove virtual servers to a fleet of application servers in response to the existing network traffic. A load balancer will also be needed to distribute traffic to those instances that run the Wordpress app across multiple AZs as they are auto scaled in and out. 

Create security group for the load balancer: 
1. WP Load Balancer SG

Must edit the inbound rules for WP Load Balancer SG and allow **HTTP** traffic on **port 80**. Select **My IP** as the source. 

Now create the load balancer: 
1. in the EC2 console select Load Balancers then select create Load Balancer
2. create an Application Load Balancer
3. give the load balancer a name and configure it to be internet facing
4. under network mapping select the VPC created and the two public subnets from each AZ
5. select the WP Load Balancer SG
6. create a target group under listeners and routing define protocol HTTP on port 80
7. name the target group without defining any targets
8. refresh the listener and the created target group should appear then select it

Configured Application Load Balancer and target group:
   
<img width="1206" alt="Load Balancer" src="https://github.com/Shereefo/AWS-Projects/assets/137960467/2f7c34bf-64e4-4376-bce7-75b7dc95621c">

This highly avaialable application server will be running PHP to use the created resources and services as a part of the scalable installation of Wordpress

Create a security group for the Wordpress servers: 
1. WP Wordpress SG

Must edit the inbound rules for WP Wordpress SG and allow **HTTP** traffic on **port 80** from the WP Load Balancer SG

Now create the **launch template** for the auto scaling group it is an instance configuration information that can be resused, shared and launched at a later time: 
1. select launch templates in the EC2 console and then create launch template
2. choose the Amazon Linux 2 AMI by selecting the right AMI ID based on the region
3. select the instance type - t2.small
4. under additional configuration expand advanced details and use the following script in the user data field:
sudo yum update -y

#install apache server
sudo yum install -y httpd
sudo amazon-linux-extras enable php7.4

#install php 
yum isntall -y php php- 
{pear,cgi,common,curl,mbstring,gd,mysqlnd,gettext,bcmanth,json,xml,fpm,intl,zip,imap,devel}

#install imagick extension
yum -y install gcc ImageMagick ImageMagick-devel Image Magick-perl 
pecl install imagick

5. update the Bash script with the values from your environment (these are my values):

#!/bin/bash

DB_NAME="wordpress"
DB_USERNAME="wpadmin"
DB_PASSWORD="dfgtvzz4"
DB_HOST="wordpress-workshop.cluster-ctdnyvvewl6s.eu-west-1.rds.amazonaws.com"

yum update -y

#install apache server 
yum install -y httpd

#install php
amazon-linux-extras enable php7.4
yum clean metadata 
yum install php php-devel
amazon-linux-extras install -y php7.4
systemctl start httpd
systemctl enable httpd

#install wordpress
cd /var/www
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php


cp -r wordpress/* /var/www/html/
sed -i "s/database_name_here/$wordpress/g" /var/www/html/wp-config.php
sed -i "s/username_here/$wpadmin/g" /var/www/html/wp-config.php
sed -i "s/password_here/$dfgtvzz4/g" /var/www/html/wp-config.php
sed -i "s/localhost/$wordpress-workshop.cluster-ctdnyvvewl6s.eu-west-1.rds.amazonaws.com/g" /var/www/html/wp-config.php
### keys update

#change httpd.conf file to allowoverride
enable .htaccess files in Apache config using sed command
sudo sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf

Change OWNER and permission of directory /var/www
chown -R apache /var/www
chgrp -R apache /var/www
chmod 2775 /var/www
find /var/www -type d -exec chmod 2775 {} \;
find /var/www -type f -exec chmod 0664 {} \;
echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php

systemctl restart httpd
systemctl enable httpd
systemctl start httpd

6. under security groups select:
- WP Cache Client SG
- WP DB Client SG
- WP EFS Client SG
- WP Wordpress SG




After creating the launch configuration create the **autoscaling group (ASG)** for the Wordpress back-end web servers: 
1. select auto scaling groups in the EC2 dashboard
2. press create an auto scaling group, create a name
3. select the previously created launch template
4. ensure your VPC is selected with the correct subnets of the web servers this should be application subnet A & B
5. configure advanced options and choose attatch to an existing load balancer
6. choose the target group created earlier from the existing load balancer target group list
7. configure the group size: Desired cap. - 2, minimum cap. - 2, maximum cap. - 4.
8. select target tracking scaling policy with the metric type - average CPU utilization
9. accept the remaining defaults to complete the creation of the ASG 

Configured auto scaling group and launch template: 
   
<img width="1224" alt="ASG" src="https://github.com/Shereefo/AWS-Projects/assets/137960467/823e5146-ebe5-42d2-bfd6-6b89a9283348">

Finally when the targets are deemed healty in the target group you can open the DNS name for the Application load balancer to view the freshly crafted Wordpress installation: 

<img width="997" alt="Wordpress" src="https://github.com/Shereefo/AWS-Projects/assets/137960467/f4893b0b-266f-4b1a-9789-e64206d78328">

I would like to give AWS workshops most of the credit for helping me follow along on this first cloud & AWS project of mine. It was a lot of fun seeing how a highly avaialble web app can come together in one of the many methods using AWS. 
