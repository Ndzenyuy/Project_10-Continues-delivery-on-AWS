# Project 10: CICD Pipeline on AWS

In project 8, I deployed a Continues integration on AWS, this project is actually a continuation of it, I will be implementing the continues delivery part of it. The code is hosted in Codecommit, whenever there is an update, Cloudwatch events triggers the pipeline which fetches the code, runs a test on Sonarcloud, if the quality gate is passed, the pipeline launches the build phase, stores the built artifact in an S3 bucket next Elastic Beanstalk picks the deployment in S3 and update its environment. After a 360 seconds waiting, the final stage of the pipeline is triggered which tests the deployed software, the credentials of a test user, initially stored in DB is used to login to the webApp and takes screenshots which are stored in S3 aswell.

## Architechture
![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/project10%20architecture.jpg)

## Technologies
  - AWS Codebuild
  - AWS codeartifact
  - AWS Codecommit
  - AWS Elastic Beanstalk
  - AWS RDS-MySQL

## Prerequisites
  - AWS account
  - Project 8

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
   ![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/launched%20successful%20environment.png)
   ![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/launched%20enviroment%20instances.png)

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

    Back to beanstalk -> configuration -> Configure instance traffic and scaling \
    Edit processes, Healthcheck to ```\login```
    Edit Stickiness, sticky sessions = true
    ![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/enter%20dashlogin.png)
    ![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/edit%20healthchecks.png)

4. Pom.xml and settings.xml files
Edit pom.xml file to have the following contents in branch cd-aws
```
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.visualpathit</groupId>
    <artifactId>vprofile</artifactId>
    <packaging>war</packaging>
    <version>v2</version>
    <name>Visualpathit VProfile Webapp</name>
    <url>http://maven.apache.org</url>
    <properties>
        <spring.version>4.2.0.RELEASE</spring.version>
        <spring-security.version>4.0.2.RELEASE</spring-security.version>
        <spring-data-jpa.version>1.8.2.RELEASE</spring-data-jpa.version>
        <hibernate.version>4.3.11.Final</hibernate.version>
        <hibernate-validator.version>5.2.1.Final</hibernate-validator.version>
        <mysql-connector.version>5.1.36</mysql-connector.version>
        <commons-dbcp.version>1.4</commons-dbcp.version>
        <jstl.version>1.2</jstl.version>
        <junit.version>4.10</junit.version>
        <logback.version>1.1.3</logback.version>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-web</artifactId>
            <version>${spring-security.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-config</artifactId>
            <version>${spring-security.version}</version>
        </dependency>

        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-validator</artifactId>
            <version>${hibernate-validator.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-jpa</artifactId>
            <version>${spring-data-jpa.version}</version>
        </dependency>

        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-entitymanager</artifactId>
            <version>${hibernate.version}</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql-connector.version}</version>
        </dependency>

        <dependency>
            <groupId>commons-dbcp</groupId>
            <artifactId>commons-dbcp</artifactId>
            <version>${commons-dbcp.version}</version>
        </dependency>

        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>jstl</artifactId>
            <version>${jstl.version}</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
	    <groupId>org.mockito</groupId>
    	    <artifactId>mockito-core</artifactId>
            <version>1.9.5</version>
            <scope>test</scope>
	</dependency>
	<dependency>
	     <groupId>org.springframework</groupId>
	     <artifactId>spring-test</artifactId>
	     <version>3.2.3.RELEASE</version>
	     <scope>test</scope>
	</dependency>
	<dependency>
	     <groupId>javax.servlet</groupId>
	     <artifactId>javax.servlet-api</artifactId>
	     <version>3.1.0</version>
	     <scope>provided</scope>
	</dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback.version}</version>
        </dependency>
        <dependency>
    	    <groupId>org.hamcrest</groupId>
    	    <artifactId>hamcrest-all</artifactId>
    	    <version>1.3</version>
    	    <scope>test</scope>
		</dependency>
	        <dependency>
		    <groupId>commons-fileupload</groupId>
		    <artifactId>commons-fileupload</artifactId>
		    <version>1.3.1</version>
		</dependency>
		 <!-- Memcached Dependency -->
		<dependency>
		    <groupId>net.spy</groupId>
		    <artifactId>spymemcached</artifactId>
		    <version>2.12.3</version>
		</dependency>
		<dependency>
		    <groupId>commons-io</groupId>
		    <artifactId>commons-io</artifactId>
		    <version>2.4</version>
	    </dependency>
	    <!-- RabbitMQ Dependency -->
	    <dependency>
	            <groupId>org.springframework.amqp</groupId>
	            <artifactId>spring-rabbit</artifactId>
	            <version>1.7.1.RELEASE</version>
	    </dependency>

	    <dependency>
	            <groupId>com.rabbitmq</groupId>
	            <artifactId>amqp-client</artifactId>
	            <version>4.0.2</version>
	   </dependency>
	   <!-- Elasticsearch Dependency-->
		<dependency>
		    <groupId>org.elasticsearch</groupId>
		    <artifactId>elasticsearch</artifactId>
		    <version>5.6.4</version>
		</dependency>
		<!-- Transport Client-->
		<dependency>
		    <groupId>org.elasticsearch.client</groupId>
		    <artifactId>transport</artifactId>
		    <version>5.6.4</version>
		</dependency>
		<!--gson -->
		<dependency>
		    <groupId>com.google.code.gson</groupId>
		    <artifactId>gson</artifactId>
		    <version>2.8.2</version>
		</dependency>
    </dependencies>
     <build>
        <plugins>
            <plugin>
                <groupId>org.eclipse.jetty</groupId>
                <artifactId>jetty-maven-plugin</artifactId>
                <version>9.2.11.v20150529</version>
                <configuration>
                    <scanIntervalSeconds>10</scanIntervalSeconds>
                    <webApp>
                        <contextPath>/</contextPath>
                    </webApp>
                </configuration>
            </plugin>
		<!-- CODE COVERAGE -->
 		    <plugin>
 		        <groupId>org.jacoco</groupId>
 		        <artifactId>jacoco-maven-plugin</artifactId>
 		        <version>0.7.2.201409121644</version>
 		        <executions>
 		            <execution>
 		                <id>jacoco-initialize</id>
 		                <phase>process-resources</phase>
 		                <goals>
 		                    <goal>prepare-agent</goal>
 		                </goals>
 		            </execution>
 		            <execution>
 		                <id>jacoco-site</id>
 		                <phase>post-integration-test</phase>
 		                <goals>
 		                    <goal>report</goal>
 		                </goals>
 		            </execution>
 		        </executions>
 		</plugin>

        </plugins>
    </build>
  <repositories>
      <repository>
        <id>codeartifact</id>
        <name>codeartifact</name>
    <url>https://visualpath-997450571655.d.codeartifact.us-east-1.amazonaws.com/maven/maven-central-store/</url>
      </repository>
    </repositories>
</project>

```

Edit settings.xml to have the following
```
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
        <server>
            <id>codeartifact</id>
            <username>aws</username>
            <password>${env.CODEARTIFACT_AUTH_TOKEN}</password>
        </server>
    </servers>
<profiles>
  <profile>
    <id>default</id>
    <repositories>
      <repository>
        <id>codeartifact</id>
    <url>https://visualpath-997450571655.d.codeartifact.us-east-1.amazonaws.com/maven/maven-central-store/</url>
      </repository>
    </repositories>
  </profile>
</profiles>
<activeProfiles>
        <activeProfile>default</activeProfile>
    </activeProfiles>
<mirrors>
  <mirror>
    <id>codeartifact</id>
    <name>visualpath-maven-central-store</name>
    <url>https://visualpath-997450571655.d.codeartifact.us-east-1.amazonaws.com/maven/maven-central-store/</url>
    <mirrorOf>*</mirrorOf>
  </mirror>
</mirrors>
</settings>
```
5. Build Job Setup
   Modify the existing build jobs (vprofile-build-artifact and vprofile-build) edit source -> branch and put cd-aws\
   Next create a build project, which will build the artifact and deploy on beanstalk
   ```
   - name: vprofile-buildAndRelease
   - source provider: AWS CodeCommit
   - Repository: vprofile-code-repo
   - Branch: cd-aws
   - OS: ubuntu
   - Runtime: Standard
   - Image: aws/codebuild/standard:5.0
   - Environment type: Linux
   - Use existing role: codebuild-v-vprofile-build-service-role (same role used in previous builds)
   - Insert build commands -> switch to editor -> Paste the following content
            buildspec file:
                version: 0.2
                env:
                parameter-store:
                    CODEARTIFACT_AUTH_TOKEN: codeartifact-token
                    dbhost: RDS-Endpoint
                    dbuser: RDSUSER
                    dbpass: RDSPASS
                phases:
                install:
                    runtime-versions:
                    java: corretto8
                    commands:
                    - cp ./settings.xml /root/.m2/settings.xml
                pre_build:
                    commands:
                    - sed -i "s/jdbc.password=admin123/jdbc.password=$dbpass/" src/main/resources/application.properties
                    - sed -i "s/jdbc.username=admin/jdbc.username=$dbuser/" src/main/resources/application.properties
                    - sed -i "s/db01:3306/$dbhost:3306/" src/main/resources/application.properties
                    - apt-get update
                    - apt-get install -y jq
                    - wget https://downloads.apache.org/maven/maven-3/3.8.8/binaries/apache-maven-3.8.8-bin.tar.gz
                    - tar xzf apache-maven-3.8.8-bin.tar.gz
                    - ln -s apache-maven-3.8.8 maven
                build:
                    commands:
                    - mvn clean install -DskipTests
                artifacts:
                files:
                    - '**/*'
                base-directory: 'target/vprofile-v2'

    - Logs: Group name = vprofile-cicd-logs
            stream name: Build&releaseJob

    - And create

   ```
   ![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/parameter%20store.png)

   Back to Terminal on your computer, regenerate the CODEARTIFACT-AUTH-TOKEN, run the following
   ```
     export CODEARTIFACT_AUTH_TOKEN=`aws codeartifact get-authorization-token --domain visualpath --domain-owner 997450571655 --region us-east-1 --query authorizationToken --output text`

    echo $CODEARTIFACT_AUTH_TOKEN
   ```
   Copy the generated token and update the contents of codeartifact-token in parameter store where the values of the former project were. \

   Create new parameters
   ```
    RDS-Endpoint: <endpoint for the rds db created earlier>
    RDSUSER: admin
    RDSPASS: <the-generated-password-during-db-creation>
   ```

6. Software testing Job
    Create s3bucket
    ```
        name: vprofile-cicd-testoutput
        region: us-east-1

    ```
    create a software testing job:
    ```
    name: SoftwareTesting
    source: aws codecommit
    repository: vprofile-code-repo
    branch: seleniumAutoScripts
    OS: windows server 2019
    runtime: base
    image: aws/codebuild/windows-base:2019-1.0
    service role: existing role-> codebuild-v-build-vprofile-role
    Buildspec: insert build command -> swith to editor and paste the following
        version: 0.2

        #env:
        #  variables:
        #    PROJECT: AspNetMvcSampleApp
        #   DOTNET_FRAMEWORK: 4.6.1
        phases:
        build:
            commands:
            - Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
            - choco install jdk8 -y
            - choco install maven -y
            - choco install googlechrome --ignore-checksum -y
            - choco install git -y
            - mkdir C:\output
            - Start-Sleep -s 360
            - cd src\seleniumscripts\
            - mvn clean test -Dsurefire  -Durl=http://pineapple-env.eba-xvrnrqp3.us-east-1.elasticbeanstalk.com/login -Dusr=admin_vp -Dpass=admin_vp -DsShotPath=C:\\output\\ScrnSht_
            - cp C:\output\* .
        artifacts:
        files:
            - '**/*'

    Artifacts: - type: amazon s3
               - bucket name: vprofile-cicd-testoutput
               - enable semantic versioning = true
               - artifact packaging: zip
    Logs: Group name = vprofile-cicd-logs
            stream name: softwareTestsLogs

    create build project

    ```
    ![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/buildprojects.png)

7. Pipeline setup
    Create a pipeline
    ```
    step 1: choose pipeline settings
        name: vprofile-cicd-pipeline
        service role: new service role
        -> next
    step 2: Add source stage
        source provider: AWS codecommit
        repository name: vprofile-code-repo
        branch: cd-aws
        -> next
    step 3: Add build stage
        build provider: AWS codebuild
        region: default displayed region
        Project name: vprofile-BuildAndRelease(created earlier)
        -> next
    step 4: Add deploy stage
        Deploy provider: AWS Elastic Beanstalk
        Region: default display
        Application name: vprofile app
        environment name: vprofileapp-env
        -> next
    step 5: create pipeline

    ```
    Stop the pipeline execution for further editing. Goto to edit, after the source stage, click add stage
    ```
        name: CodeAnalysis
        stage: new stage
            action name: CodeAnalysis
            Action provider: AWS CodeBuild
            input artifact: sourceArtifact
            Project name: vprofile-build
            -> done
    ```
    We create another source stage after CodeAnalysis
    ```
        name: BuildAndStore
        stage: new stage
            action name: BuildAndStore
            Action provider: AWS CodeBuild
            input artifact: sourceArtifact
            Project name: vprofile-build-artifact
            output artifacts: buildArtifact
            -> done
    ```

    we add another stage after buildAndStore
    ```
        name: DeployToS3
        stage: new stage
            action name: DeployToS3
            Action provider: Amazon s3
            input artifact: buildArtifact
            Bucket: vprofile-artifact-storage
            extract file before deploy = true
            -> done
    ```

    Edit the build job by changing the following
    ```
    output artifacts = BuildArtifactToBean -> done
    ```

    Edit the deploy job, change
    ```
    input artifact = BuildArtifactToBean -> done
    ```

    Add a stage at the end of the pipeline
    ```
    name: SoftwareTesting
    stage: new stage
        action name: SoftwareTesting
        Action provider: AWS CodeBuild
        input artifact: SourceArtifact
        Project name: SoftwareTesting
        -> done

    ```
Save pipeline, and release change to launch the pipeline

![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/successful-pipeline.png)

Update notifications
```
manage notifications rules
create notification
notification rule settings
    name: vprofile-aws-cicd-pipeline-notification
    detail type: full
    event that trigger notification: select all
    targets: vprofile-pipeline notification
    -> submit
```
![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/login-page.png)
![[](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/successful-login.png)

When pipeline is successful, sofware testing delivers a zip folder in the bucket we created. When downloaded, we can see a screenshot of the login page, to show the software is successful
![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/Screenshot%20from%202023-08-31%2013-23-00.png)
![](https://github.com/Ndzenyuy/Project_10-Continues-delivery-on-AWS/blob/cicd-aws/images/Screenshot%20from%202023-08-31%2013-23-40.png)

