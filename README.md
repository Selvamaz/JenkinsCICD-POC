# Jenkins-CICD-POC

![Path](https://github.com/Selvamaz/Jenkins-CICD-POC/assets/158442659/3d2655aa-c2d1-46b1-9450-87a6b9dc43c4)


<br>

> [!Note]
> **Tools Used : Github, Maven, Jenkins, Sonarqube, Postgres, AWS RDS, Splunk and Docker**


## Basic
1. Create two instance (t2.medium, 12GB HDD, Ubuntu 22 and All port allowed SG). One for Jenkins pipeline and one for sonarqube (sonarqube requires 4gb ram and dual core, so for now it needs separate server)
2. RDS -> Create subnet group with all 3 zones in ap-south-1
3. Create a postgres RDS database (Free tier, use above subnet group, public (for now), user : postgres, pass:"Password", all port allowed security group)
4. Login into both servers, install same java version (here we installing openjava11), zip
5. Check java installed or not using ``` java —version``` 
6. Export java environment variables 0> check java path usage ``` sudo update-alternatives —config java```
    ```
    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64/```  (don’t include bin and java)
    export ES_JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64/```  (don’t include bin and java)
    export PATH=$PATH:$JAVA_HOME/``` 

## Jenkins Server Preparations 
1. Install Jenkins using official documentation (after done, check jenkins —version)
2. Install docker using official documentation (after done, check docker —version)
3. Install maven (use the latest version)
    ```
    wget https://dlcdn.apache.org/maven/maven-3/3.9.7/binaries/apache-maven-3.9.7-bin.zip
    unzip apache-maven-3.9.7-bin.zip
    mv  apache-maven-3.9.7 /opt/maven``` 
5. Turn maven into an systemctl executable 
   ```
   export M2_HOME="/opt/maven"``` 
   export PATH="$M2_HOME/bin:$PATH"``` 
   echo $PATH``` 
7. Pls check ``` mvn -v```  or ```maven —version``` 
8. Install GIT ```sudo apt-get install git``` 

## Jenkins Server ``` http://server-public-ip:8080``` 
1. Jenkins Server : ``` cat /var/lib/jenkins/secrets/initialAdminPassword```  and copy the code and paste in jenkins dashboard. Install recommended plugins)
2. Create new pipeline project and tick its a GitHub project. Past the GitHub link
3. Copy the first stage : Bringing GitHub project inside jenkins (pull or fork)
```   
node{
    stage('SCM Checkout'){
        git 'https://github.com/Selvamaz/DemoCIApp.git'
   }
}
```
5. Save -> Build 
6. Jenkins -> Manage Jenkins -> Tools -> Add Maven Installations -> Give a name(example MavenTool, will be using in script) -> give the path /opt/maven -> Save
7. Copy the second stage inside node part (node is the script start and end). This script bring MavenTool inside, second line make the package and rename the package as newapp.war
``` 
stage('Compile-Package'){
       def mvnHome =  tool name: 'MavenTool', type: 'maven'   
       sh "${mvnHome}/bin/mvn clean package"
       sh 'mv target/myweb*.war target/newapp.war'
   }
``` 
8. Make sure this script is inside node {} and save -> Build

## Sonarqube Server Installations
1. Install postgres ``` sudo apt install postgresql``` 
2. login to postures using : ``` psql -h sonarqubedb.c4gge0fgnzfq.ap-south-1.rds.amazonaws.com -U postgres```  -> Enter password and log in to the database
3. Execute below commands by one by one inside postures
    a. ``` CREATE USER sonar;
    b. ALTER USER sonar WITH ENCRYPTED PASSWORD '"Password"’;
    c. CREATE DATABASE sonarqube;
    d. ALTER DATABASE sonarqube OWNER TO sonar;
    e. GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;``` 
4. Install sonarqube latest LTS version. Now 9.9. ``` wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.1.69595.zip```  -> unzip -> move to /opt/sonarqube
5. ``` sudo vi /opt/sonarqube/conf/sonar.properties```  (give rds username, rds password, postgres link : jdbc:postgress://rds_enpoing:port_number/
6. In the same file, un # sonar.web.port, sonar.web.host and sonar.web.context(if u want path like www.google.com/map)
7. Exit from root -> sudo chown -R ubuntu:ubuntu /opt/sonarqube/ (ubuntu is the ubuntu username) -> ``` sudo sysctl -w vm.max_map_count=300000```  (increase vm size for sonarqube)
8. Execute ``` sh /opt/sonarqube/bin/linux-x86-64/sonar.sh start``` 
9. Check Status ``` sh /opt/sonarqube/bin/linux-x86-64/sonar.sh status``` 
10. Once running -> ``` http://sonar_server_ip:9000```  -> admin, admin
11. Account -> Security -> Create token for maven
12. Make it running in the background

## Back to Jenkins Server
1. Install sonarqube scanner in Jenkins. Add sonarqube token to credentials as a secret text & CredentialsID
2. Manage Jenkins -> System -> Add Sonar -> Give a sonar config name “SonarQube” and path as /opt/sonar
3. Copy the third stage: MavenTool is maven config name and SonarQube is sonar config name
``` 
stage('SonarQube Analysis'){
       def mvnHome =  tool name: 'MavenTool', type: 'maven'
       withSonarQubeEnv('SonarQube'){
           sh "${mvnHome}/bin/mvn sonar:sonar"
       }
   }
```  
4. Give jenkins permission to access docker by ``` sudo chmod 777 ///var/run/docker.sock```  
5. Build docker image. Whole project is copied to the location and jenkins project runs the build in that directory. So . refers the current directory and holds the Dockerfile
```
    stage('Build Docker Imager'){
       sh 'docker build -t selvamax/myweb:0.0.2 .'
   }
``` 
6. Login to docker : docker login
7. Save docker login password as secret text credential (copy the credential ID, here DockerPWD), variable is the new variable name and we can use any name
8. Copy the next stage pushing the image to docker hub
``` 
stage('Docker Image Push'){
       withCredentials([string(credentialsId: 'DockerPWD', variable: 'DockerPWD')]) {
           sh "docker login -u selvamax -p ${DockerPWD}"
       }
   }
``` 
9. We will building many containers, so we have to remove old containers and give path the new images. So the scrip to remove old containers. Docker won’t replace old one. tomcattest is the container name
``` 
stage('Remove Previous Container'){
       try{
           sh 'docker rm -f tomcattest'
       }catch(error){
           //  do nothing if there is an exception
        }
   }
``` 
10. Create container deployment from docker image hub of ours
``` 
stage('Docker deployment'){
       sh 'docker run -d -p 8090:8080 --name tomcattest selvamax/myweb:0.0.2' 
   }
``` 
11. Hit ``` http://jenkins_server_public_ip:tomcat_port(here 8090)/newapp```  (newapp is the app name we build as a war)
12. docker ps and docker images to confirm the image and container
13. Turn on web hooks in jenkins 
14. Our projet GitHub repo -> settings -> web hooks -> JSON -> http://jenkins_public:port/github-webook/
15. Thats all

## Application Monitoring using Splunk
1. Create a EC2 server with all ports enabled (for demo purposes) and give 30GB space. Its Master
2. Choose Enterprise Version - Splunk -> wget -> Tar -> Move to /opt/splunk
3. ```./splunk start --accept-license``` -> credentials -> admin -> admin123
4. Browser -> http://master_public_ip:8000 -> admin, admin -> login

## Back to Application Deployed Server - [Jenkin Server] 
<blink>*** Dont go inside docker and install ***</blink><br>

5. *** Goto ROOT *** -> Install Universal Forwarder in slave machine (same version as splunk)->  wget -> Tar -> Move /opt/universalforwarder<br>
6. ```./splunk start —accept-license``` -> admin, admin123<br>
7. Copy ```cd /var/lib/docker/container/container_id``` (here docker saves log of applications like httpd, tomcat)<br>
8. Splunk Default Listen port (9997) -> ```./splunk add forward-server master_ip_address:9997``` (no http -> we can use any port number but we have to use same)<br>
9. Add logs to indexer -> ```./splunk add monitor /var/lib/docker/container/container_id/ -index main -sourcetype hdfclogs (index name can any one but remember)<br>

## Back to Splunk Master Server
10. Splunk Web UI -> Settings -> Forwarding and Receiving -> Receive Data -> Configure Receiving -> Add 9997 (above mentioned) -> Save
12. Home -> search and reporting -> Data Summary -> Receives Data

## Thanks all Folks !!!
