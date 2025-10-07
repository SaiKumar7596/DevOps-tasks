# 🚀 End-to-End: Build Java (Maven) on EC2 (Server-1) → Deploy WAR to Tomcat on EC2 (Server-2)

> Purpose: A complete, copy‑paste friendly guide from creating EC2 instances to successful deployment of a `.war` built with Maven on one server and deployed to Apache Tomcat on a second server. Ready to drop into a GitHub repo as `README.md` or `DEPLOYMENT_GUIDE.md`.

What it is: A straight‑forward runbook to build a Java web application with Maven on a build EC2 instance and deploy the resulting WAR to an Apache Tomcat server on a separate EC2 instance.

Why it matters: Separating build and runtime servers keeps CI/CD pipelines clean, improves security (build environment does not serve traffic), and mirrors production deployment patterns used in real-world teams.


Step-by-step
> These commands assume Ubuntu servers and the `ubuntu` user. Adapt for other distros/users.

### A. Create EC2 instances (Console or CLI)
Via AWS Console (recommended for first time):**
1. EC2 → Launch instance → Select Ubuntu Server 22.04 LTS (or Amazon Linux 2).
2. Choose instance type (t2.micro for testing, or t3.small+ for builds).
3. Configure security group:
   - Allow SSH (TCP 22) from your IP
   - Allow HTTP (TCP 8080) from wherever you access Tomcat (or 0.0.0.0/0 for testing)
4. Add your existing key pair or create new and download `.pem`.
5. Launch two instances: `build-server` and `deploy-server`.


Once on the build-server (Ubuntu example):

# Update OS
```
sudo apt update && sudo apt upgrade -y
```
# Install Java, Maven and Git
```
sudo apt install -y openjdk-11-jdk maven git
```
# Verify
```
java -version
mvn -version
```

Have created own code manuallu, and the architecture is:
javawebapp/
├── pom.xml
└── src/
    └── main/
        ├── java/
        ├── resources/
        └── webapp/
            └── WEB-INF/web.xml
pom.xml
```
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>javawebapp</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>

    <name>Java WebApp Example</name>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- Provided by GlassFish or JBoss -->
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>5.0.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <finalName>javawebapp</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.3.2</version>
                <configuration>
                    <failOnMissingWebXml>false</failOnMissingWebXml>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```
Create the file:
```
src/main/java/com/example/web/HelloServlet.java
```
paste in HelloServlet.java below code
```
package com.example.web;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import java.io.IOException;

public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws IOException {
        response.setContentType("text/html");
        response.getWriter().println("<h1>Hello from Java WebApp!</h1>");
        response.getWriter().println("<p>Deployed on " + request.getServerName() + "</p>");
    }
}
```
Create web.xml
```
src/main/webapp/WEB-INF/web.xml
```
And paste the below code in web.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee
         https://jakarta.ee/xml/ns/jakartaee/web-app_5_0.xsd"
         version="5.0">

    <servlet>
        <servlet-name>HelloServlet</servlet-name>
        <servlet-class>com.example.web.HelloServlet</servlet-class>
    </servlet>

    <servlet-mapping>
        <servlet-name>HelloServlet</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>
</web-app>
```
Add Index.JSP
```
src/main/webapp/index.jsp
```
And paste code in web.xml
```
<html>
  <head><title>Java WebApp</title></head>
  <body>
    <h2>Welcome to Java WebApp!</h2>
    <p><a href="hello">Go to HelloServlet</a></p>
  </body>
</html>
```
Build the WAR artifact:
```
mvn clean package
# After build find the WAR
ls -l target/*.war
# e.g. target/webapp-0.1.3.war
```

If the build produces `webapp-0.1.3.war`, note the full path: `/home/ubuntu/JavaWebCalculator/target/webapp-0.1.3.war`


# C. Install JBoss (WildFly) (Deploy Server)
SSH into Tomcat server:
```bash
ssh -i task-key.pem ubuntu@18.232.137.143
```

Install Java (on Deploy server):
```bash
sudo apt update && sudo apt install -y openjdk-11-jdk wget
java -version
```

Download and Extract:
```
cd /home/ubuntu
wget https://github.com/wildfly/wildfly/releases/download/30.0.0.Final/wildfly-30.0.0.Final.zip
unzip wildfly-30.0.0.Final.zip
mv wildfly-30.0.0.Final wildfly
```
Start WildFly:
```
cd wildfly/bin
./standalone.sh &
```
Deploy WAR:
```
scp -i task-key.pem /home/ubuntu/javawebapp/target/javawebapp.war ubuntu@52.207.229.61:/home/ubuntu/wildfly/standalone/deployments/
```

Make Jboss scripts executable and create a dedicated tomcat user (recommended):

Open port 8080 in Deploy Server on console(edit inbound rules)

Start Jboss  (manual):
```
bin/startup.sh
# start the server
./startup.sh
```

Check Tomcat logs to confirm it started:
```bash
tail -n 50 /opt/apache-tomcat-9.0.109/logs/catalina.out
```


### D. Transfer the WAR from Build Server → Tomcat Server
On your **local machine** you probably copied the PEM to build server earlier. If not, copy PEM carefully and keep permission 400:
To copy pem file:
```
scp -i task-key.pem task-key.pem username@<ip of build server>:~

```bash
# (on local) ensure pem has right perms
chmod 400 task-key.pem
```

From the **build server**, use `scp` to copy the WAR into Tomcat's `webapps` directory. Example you already used:
```bash
scp -i task-key.pem /home/ubuntu/JavaWebCalculator/target/webapp-0.1.3.war \
    ubuntu@18.232.137.143:~/apache-tomcat-9.0.109/webapps/
```

Example: copy to home then move with sudo:
```bash
# From build-server
scp -i task-key.pem target/webapp-0.1.3.war ubuntu@18.232.137.143:/home/ubuntu/



E. Deploy & Verify
1. Tail the logs while deploying:
```bash
# on tomcat server
tail -f /opt/apache-tomcat-9.0.109/logs/catalina.out
```

2. Visit the app in browser:
```
http://18.232.137.143:8080/webapp-0.1.3/
```
(If your WAR file name is `webapp-0.1.3.war` the context path becomes `/webapp-0.1.3`.)

---


## 7) Troubleshooting
- **SCP permission denied**: ensure `chmod 400 task-key.pem` on machine where `scp` runs and correct user (`ubuntu`).
- **404 after deploy**: check `catalina.out` for deployment exceptions. Ensure WAR is not corrupted (`unzip -t webapp-0.1.3.war`).
- **App fails to start**: check Java version and missing libraries.
- **Port 8080 blocked**: confirm AWS Security Group and VM firewall (ufw/firewalld) allow 8080.
- **Permission issues moving WAR**: if Tomcat owned by `tomcat` user, copy to `/home/ubuntu` then `sudo mv` into `/opt/...` as shown in script.
- **Do not commit PEM**: never store `*.pem` in git. Use `.gitignore` and secrets manager.

## 9) Appendix: Quick reference commands
- Start/stop Tomcat (manual):
  - `bin/startup.sh`
  - `bin/shutdown.sh`
- Tail logs: `tail -f /opt/apache-tomcat-9.0.109/logs/catalina.out`
- Find WAR: `ls -l target/*.war`
- Java & Maven: `java -version`, `mvn -version`
- SCP example:
```bash
scp -i task-key.pem /path/to/webapp-0.1.3.war ubuntu@18.232.137.143:/home/ubuntu/
```


**End of document.**

*You can copy this file into `README.md` in your GitHub repo or save as `docs/DEPLOYMENT_GUIDE.md`.*

