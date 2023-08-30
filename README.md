# Project 10: CICD Pipeline on AWS

In project 8, I deployed a Continues integration on AWS, this project is actually a continuation of it, I will be implementing the continues delivery part of it. The code is hosted in Codecommit, whenever there is an update, Cloudwatch events triggers the pipeline which fetches the code, runs a test on Sonarcloud, if the quality gate is passed, the pipeline launches the build phase, stores the built artifact in an S3 bucket next Elastic Beanstalk picks the deployment in S3 and update its environment. After a 360 seconds waiting, the final stage of the pipeline is triggered which tests the deployed software, the credentials of a test user, initially stored in DB is used to login to the webApp and takes screenshots which are stored in S3 aswell.

## Architechture
![](project10-architecture)

## Technologies
  - AWS Codebuild
  - AWS codeartifact
  - AWS Codecommit
  - AWS Elastic Beanstalk
  - AWS RDS-MySQL

## Prerequisites
  - AWS account
  
## Steps
1. Beanstalk setup \
   Create a beanstalk environment
 ```step 1: Configure Environment
        Application: 
        Application Name: vprofile-app
        Plateform: Tomcat
            Platform branch: Tomcat 8.5 with corretto 8 running on 64bit Amazon linux 2
            Platform version: 4.3.10
        Application Code: Sample application
        Presets: Single Instance

    step 2: Configure Service Access
        Service role: create and use new service role
        choose EC2-keypair
        EC2 instance profile: s3fullAccess

    step 3:  setup networking, database and tags
        VPC: default
        Public IP address: activated=true
        Instance subnets: <select-azs: 1a,1b,1c>

    step 4: Configure instance traffic and scaling
        Secrurity group: create security group with inbounds rule port 8080 and 80 on all ipv4
        Autoscaling group: Environment type -> Load Balanced
        Instances: min=2, max=4
        Instance type: t2.micro
        processes: select default and edit stickness -> enable sticky sessions   
    
    step 5: Configure updates, monitoring and logging: leave defaults

    step 6: Review and submit

   ```
   ![](Created beanstalk environment)
   ![](launched instances)

2. RRDS Setup
```
    - Create database:
      -Standard create
      - Mysql
      - version 5.7
      - name: vprofile-cicd-project
      - autogenerate password=true
      - single AZ
      - create secgroup: vprofile-cicd-mysql-sg
      - Additional configurations: 
        initial database name: accounts
     -> create database
     -> load credentials and save them(username and password)
```

3. Security Groups and RDS initializations \
    On the AWS console goto EC2, check the security group of one of the running instances and copy the security group id, edit inboud rule of the database create a rule"
    ```allow Mysql traffic from EC2-secGroup-id```
    Now ssh into one of the EC2 instances managed by elastic beanstalk, we have to connect to the DB and initialise it:
    ```
    sudo -i
    yum install mysql git -y
    mysql -h <RDS-mysql-endpoint> -u admin -p<the-Auto-generated-Password-During-Creation> 
    ```
    If connection is successful, then we are good to go. Next
    ```
    git clone https://github.com/Ndzenyuy/vprofile-project1.git
    cd vprofile-project1
    git checkout cd-aws
    cd src/main/resources/
    mysql -h <RDS-mysql-endpoint> -u admin -p<the-Auto-generated-Password-During-Creation> < db_backup.sql

    ```

    Back to beanstalk -> configuration -> Configure instance traffic and scaling
 
4. Pom.xml and settings.xml files
5. Build Job Setup
6. Software testing Job
7. Pipeline setup
