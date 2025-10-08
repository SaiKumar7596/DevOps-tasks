#  Java Web Application: Build & Deploy Guide (Java 17 + Maven + Tomcat)


## Overview
This project is a **Java Web Application** designed to run on **Java 17**, built using **Maven**, and deployed on **Apache Tomcat**. The project demonstrates a basic Java web app structure, with a simple calculator implementation.


#  Prepare Build Server

<img width="602" height="260" alt="image" src="https://github.com/user-attachments/assets/92246703-330e-40f3-9e4b-7934b05ccdf1" />

<img width="602" height="59" alt="image" src="https://github.com/user-attachments/assets/907da388-2d8b-4d54-9f64-4a7bd1e370aa" />


## 1. Install prerequisites 

update + essential tools 
```
sudo apt update 
sudo apt install -y openjdk-17-jdk maven git wget unzip curl 
sudo apt install -y maven 
 ```
## verify 
```
java -version 
mvn -version 
```
<img width="602" height="157" alt="image" src="https://github.com/user-attachments/assets/65c767c2-b964-4abc-81de-38bf3dd82083" />



Use openjdk-11 if you need Java 11; adjust accordinngly.

## 2. Get the project source 

If your project is in Git: 

Git will be pre installed in the linux surver if nit install git using sudo apt –y install git than : 

```
cd ~ git clone https://github.com/akracad/JavaWebCal.git 
```
<img width="602" height="129" alt="image" src="https://github.com/user-attachments/assets/1926a46d-923f-487c-b995-a5cb4c287731" />

 

If the code is already on the build server, cd into the project directory: 

cd /home/ubuntu/JavaWebCalculator 


## 3. Check or update pom.xml (recommended) 
Ensure maven-war-plugin is modern version as when u install maven it installs the latest version: 

```
<build> 
  <plugins> 
    <plugin> 
      <groupId>org.apache.maven.plugins</groupId> 
      <artifactId>maven-war-plugin</artifactId> 
      <version>3.4.0</version> 
    </plugin> 
  </plugins> 
</build> 
```

Confirm maven-compiler-plugin target/source are correct for your Java version. 

## 4. Build the WAR from project root 

``` mvn clean package ```

<img width="602" height="106" alt="image" src="https://github.com/user-attachments/assets/77df569e-48c3-41c7-a3de-846f5286ccd7" />


maven comand will only wrok inside the project directory.
 
## after success check 

``` ls -l target/*.war ```

<img width="602" height="44" alt="image" src="https://github.com/user-attachments/assets/13385bcc-15bf-4f9d-b356-91149a5be994" />


# Prepare App Deploy Server (Tomcat)

In security group allow port 8080 in inbound rules while creating the deploy server.

<img width="602" height="260" alt="image" src="https://github.com/user-attachments/assets/b6f51d52-93e5-4042-bf4c-0aa5df4514ce" />


## 1. Install Java (if not present) 

```
sudo apt update 
sudo apt install -y openjdk-17-jre-headless 
java -version

```
<img width="602" height="44" alt="image" src="https://github.com/user-attachments/assets/e79f4081-2cd1-4b76-9e62-ccef59032441" />


## 2. Install Tomcat on Deploy server 

``` cd /home/ubuntu ```

``` wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.109/bin/apache-tomcat-9.0.109.tar.gz ```

<img width="602" height="113" alt="image" src="https://github.com/user-attachments/assets/9bb33f23-742a-427c-b48c-0fd8b613e6b8" />


To extract the tar file 

``` tar -xvzf apache-tomcat-9.0.109.tar.gz ```

<img width="602" height="97" alt="image" src="https://github.com/user-attachments/assets/d00d4d0f-7575-42e0-862e-138ff272e1d2" />


## make scripts executable 

``` sudo chmod +x /opt/tomcat/bin/*.sh ```
 
## shutdown 

```
Cd /opt/tomcat/bin/ 
./shutdown.sh
```
<img width="602" height="170" alt="image" src="https://github.com/user-attachments/assets/2a9ecc3b-52af-48f3-9146-960ff968e123" />

 
## start 

```
Cd /opt/tomcat/bin/ 
./startup.sh 

```
<img width="602" height="170" alt="image" src="https://github.com/user-attachments/assets/01496dfd-f556-4a20-b51c-3b9b700ac235" />

## 3. Configure Tomcat Manager (if you want browser-managed deployments) 
Edit : vi /opt/tomcat/conf/tomcat-users.xml  and add a manager user: 
```
<role rolename="manager-gui"/> 
<role rolename="manager-script"/> 
<user username="admin" password="admin" roles="manager-gui,manager-script"/> 
```
<img width="602" height="169" alt="image" src="https://github.com/user-attachments/assets/9796a250-988b-4a54-bb2a-2e68b0aa9317" />

## 4. Edit the context.xml
Follow below images and edit both ./webapp/manager/META-INF/context.xml and  ./webapp/host-manager/META-INF/context.xml and remove 127.0.0.0 path.

<img width="602" height="59" alt="image" src="https://github.com/user-attachments/assets/3892d7ca-178c-4774-a7b6-081f2aec1e5f" />

<img width="602" height="87" alt="image" src="https://github.com/user-attachments/assets/0d83212c-a3aa-4aec-bcaf-aea23f9c5918" />



# Transfer the WAR from Build server to Deploy Server

<img width="602" height="319" alt="image" src="https://github.com/user-attachments/assets/ccb5e953-7c32-4d61-8fa1-019b243e0950" />



## scp using an SSH key local mechine Copy the SSH Key to Build Server
 

```
scp -i /path/to/local-key.pem /path/to/kk.key.pem ubuntu@BUILD_SERVER_IP:~/ 
```

<img width="602" height="151" alt="image" src="https://github.com/user-attachments/assets/94683e0a-d1c7-4022-897a-00da7781702c" />

<img width="602" height="151" alt="image" src="https://github.com/user-attachments/assets/42ef8d9b-cdbc-4c60-abea-168bace61bea" />

## Change the .pem file permission:


<img width="584" height="226" alt="image" src="https://github.com/user-attachments/assets/5134545e-c4cc-483c-8b53-b07e7075ed12" />

## Transfer WAR from Build Server to Deploy Server
 
scp -i /path/to/SSH_KEY target/${WAR_NAME} ${DEPLOY_USER}@${DEPLOY_HOST}:${TOMCAT_HOME}/webapps/ 

Example substitution: 

scp -i kk.key.pem /home/ubuntu/JavaWebCalculator/target/webapp-0.1.3.war ubuntu@13.222.165.210:~/apache-tomcat-9.0.109/webapps/

<img width="602" height="48" alt="image" src="https://github.com/user-attachments/assets/c6127bf0-697c-45ec-95cf-b9707627d057" />

## After copying, SSH to deploy server confirm the file is present


``` Ls /apache-tomcat-9.0.109/webapps/  ```
You should see webapp-0.1.3.war in that path

 
# Deploy & Restart Tomcat 

1. If Tomcat auto-deploys (default) 
Copying the WAR into $TOMCAT_HOME/webapps/ is enough. Tomcat will unpack and deploy the WAR automatically. 
2. If Not just Restart the Tomcat

``` 
cd /home/ubuntu/apache-tomcat-9.0.109/bin 
./shutdown.sh  
./startup.sh

```

# Verify app is running

Open: http://${DEPLOY_HOST}:8080/ 

Example :  http://174.129.76.14:8080

<img width="602" height="172" alt="image" src="https://github.com/user-attachments/assets/4e82cbb3-c974-4a54-9c71-44a408564d85" />

<img width="602" height="134" alt="image" src="https://github.com/user-attachments/assets/b8dce531-e091-4dc7-a8f1-55008fc8e2c8" />

<img width="602" height="195" alt="image" src="https://github.com/user-attachments/assets/e21aed10-34b1-4685-bc81-bc56e545edaf" />













 
