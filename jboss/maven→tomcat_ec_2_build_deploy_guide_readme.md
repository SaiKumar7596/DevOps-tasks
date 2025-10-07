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
sudo apt update && sudo apt upgrade -y

# Install Java, Maven and Git
sudo apt install -y openjdk-11-jdk maven git

# Verify
java -version
mvn -version
```

Clone your project from GitHub:
```bash
cd /home/ubuntu
git clone https://github.com/<your-username>/<your-repo>.git JavaWebCalculator
cd JavaWebCalculator
```

Build the WAR artifact:
```bash
mvn clean package
# After build find the WAR
ls -l target/*.war
# e.g. target/webapp-0.1.3.war
```

If the build produces `webapp-0.1.3.war`, note the full path: `/home/ubuntu/JavaWebCalculator/target/webapp-0.1.3.war`


# C. Prepare Tomcat Server (Server-2)
SSH into Tomcat server:
```bash
ssh -i task-key.pem ubuntu@18.232.137.143
```

Install Java (on Tomcat server):
```bash
sudo apt update && sudo apt install -y openjdk-11-jdk wget
java -version
```

Download and extract Tomcat 9 (if you already did this, skip to start steps):
```bash
cd /home/ubuntu
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.109/bin/apache-tomcat-9.0.109.tar.gz
tar xzf apache-tomcat-9.0.109.tar.gz
# Move to /opt for system-wide use (optional)
sudo mv apache-tomcat-9.0.109 /opt/apache-tomcat-9.0.109
# (If you prefer user's home) keep at ~/apache-tomcat-9.0.109
```

Make Tomcat scripts executable and create a dedicated tomcat user (recommended):
```bash
# create tomcat user (no shell)
sudo useradd -r -m -U -d /opt/apache-tomcat-9.0.109 -s /bin/false tomcat || true
sudo chown -R tomcat:tomcat /opt/apache-tomcat-9.0.109
sudo chmod +x /opt/apache-tomcat-9.0.109/bin/*.sh
```

If you prefer running as `ubuntu` (for quick testing), ensure correct permissions on webapps directory:
```bash
# if Tomcat is in home dir
# chown -R ubuntu:ubuntu ~/apache-tomcat-9.0.109
```

Open port 8080 in this server (if ufw is active):
```bash
sudo ufw allow 8080/tcp
sudo ufw reload
```

Start Tomcat (manual):
```bash
# If installed in /opt
/opt/apache-tomcat-9.0.109/bin/startup.sh
# or from user's home
~/apache-tomcat-9.0.109/bin/startup.sh
```

Check Tomcat logs to confirm it started:
```bash
tail -n 50 /opt/apache-tomcat-9.0.109/logs/catalina.out
```

**(Optional) Create a systemd service for Tomcat**
> Useful for production so Tomcat starts on boot and can be `systemctl restart tomcat`.

Create `/etc/systemd/system/tomcat.service` with appropriate content (adapt paths):
```ini
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
Environment=CATALINA_PID=/opt/apache-tomcat-9.0.109/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/apache-tomcat-9.0.109
Environment=CATALINA_BASE=/opt/apache-tomcat-9.0.109
ExecStart=/opt/apache-tomcat-9.0.109/bin/startup.sh
ExecStop=/opt/apache-tomcat-9.0.109/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

Then enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable tomcat
sudo systemctl start tomcat
sudo systemctl status tomcat
```


### D. Transfer the WAR from Build Server → Tomcat Server
On your **local machine** you probably copied the PEM to build server earlier. If not, copy PEM carefully and keep permission 400:

```bash
# (on local) ensure pem has right perms
chmod 400 task-key.pem
```

From the **build server** (or local machine), use `scp` to copy the WAR into Tomcat's `webapps` directory. Example you already used:
```bash
scp -i task-key.pem /home/ubuntu/JavaWebCalculator/target/webapp-0.1.3.war \
    ubuntu@18.232.137.143:~/apache-tomcat-9.0.109/webapps/
```

**Notes:**
- If Tomcat is under `/opt/apache-tomcat-9.0.109`, use that path instead of `~/apache-tomcat-...`.
- If you run Tomcat as user `tomcat`, you might need to `scp` to `/home/ubuntu/` and then `ssh` to move file with `sudo mv` to `/opt/...` (or `chown` appropriately).

Example: copy to home then move with sudo:
```bash
# From build-server
scp -i task-key.pem target/webapp-0.1.3.war ubuntu@18.232.137.143:/home/ubuntu/

# On tomcat server
ssh -i task-key.pem ubuntu@18.232.137.143
sudo mv /home/ubuntu/webapp-0.1.3.war /opt/apache-tomcat-9.0.109/webapps/
sudo chown tomcat:tomcat /opt/apache-tomcat-9.0.109/webapps/webapp-0.1.3.war
```

Tomcat auto-deploys WARs placed in `webapps/` if `unpackWARs` is enabled in `conf/server.xml` (default true). If Tomcat is running, it will detect the new WAR and explode the directory.


### E. Deploy & Verify
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

3. If you get 404:
- Check `catalina.out` for exceptions (class not found, config errors)
- Ensure WAR is fully copied and exploded under `webapps/webapp-0.1.3/`
- If Manager app is enabled you can also use Tomcat Manager to list deployments

4. If you changed Tomcat ownership/permissions: restart Tomcat to be safe:
```bash
# if systemd
sudo systemctl restart tomcat
# or
/opt/apache-tomcat-9.0.109/bin/shutdown.sh && /opt/apache-tomcat-9.0.109/bin/startup.sh
```

---

## 5) Automation script (copy/paste)
Create `build_and_deploy.sh` on **build server** to automate build + scp + remote deploy.

```bash
#!/bin/bash
set -e

# Variables - edit me
BUILD_DIR="/home/ubuntu/JavaWebCalculator"
WAR_NAME="webapp-0.1.3.war"
PEM="/home/ubuntu/task-key.pem"
TOMCAT_USER="ubuntu"
TOMCAT_HOST="18.232.137.143"
REMOTE_TMP_PATH="/home/ubuntu"
TOMCAT_WEBAPPS="/opt/apache-tomcat-9.0.109/webapps"

# Build
cd "$BUILD_DIR"
mvn clean package

# Transfer
scp -i "$PEM" "$BUILD_DIR/target/$WAR_NAME" ${TOMCAT_USER}@${TOMCAT_HOST}:$REMOTE_TMP_PATH/

# Deploy (move into tomcat webapps and set correct owner)
ssh -i "$PEM" ${TOMCAT_USER}@${TOMCAT_HOST} <<EOF
sudo mv $REMOTE_TMP_PATH/$WAR_NAME $TOMCAT_WEBAPPS/
sudo chown -R tomcat:tomcat $TOMCAT_WEBAPPS/$(basename $WAR_NAME .war)
sudo systemctl restart tomcat || ( $TOMCAT_WEBAPPS/bin/shutdown.sh && $TOMCAT_WEBAPPS/bin/startup.sh )
EOF

echo "Deployment finished: http://$TOMCAT_HOST:8080/$(basename $WAR_NAME .war)/"
```

Make it executable:
```bash
chmod +x build_and_deploy.sh
./build_and_deploy.sh
```

---

## 6) Add this documentation to GitHub (commands)
On your local machine or a documentation workspace:
```bash
mkdir maven-tomcat-deploy && cd maven-tomcat-deploy
# Create README.md - paste this file's content here
# or save as DEPLOYMENT_GUIDE.md inside docs/

git init
git add README.md
git commit -m "chore(docs): add Maven->Tomcat EC2 deployment guide"

# Create remote on GitHub (via web or gh CLI), then add remote:
git remote add origin git@github.com:<your-username>/maven-tomcat-deploy.git
git branch -M main
git push -u origin main
```

**Important:** Add `.gitignore` to prevent committing PEMs or build artifacts:
```
*.pem
target/
*.war
logs/
```

---

## 7) Troubleshooting & common pitfalls
- **SCP permission denied**: ensure `chmod 400 task-key.pem` on machine where `scp` runs and correct user (`ubuntu`).
- **404 after deploy**: check `catalina.out` for deployment exceptions. Ensure WAR is not corrupted (`unzip -t webapp-0.1.3.war`).
- **App fails to start**: check Java version and missing libraries.
- **Port 8080 blocked**: confirm AWS Security Group and VM firewall (ufw/firewalld) allow 8080.
- **Permission issues moving WAR**: if Tomcat owned by `tomcat` user, copy to `/home/ubuntu` then `sudo mv` into `/opt/...` as shown in script.
- **Do not commit PEM**: never store `*.pem` in git. Use `.gitignore` and secrets manager.

---

## 8) Best practices
- Use CI (Jenkins/GitHub Actions) to build and publish artifacts to an artifact repository (Nexus/Artifactory) instead of SCP.
- Create a Tomcat systemd service for stable lifecycle management.
- Use environment-specific configuration (not hard-coded) and externalize secrets.
- Use HTTPS and a reverse proxy (Nginx) in front of Tomcat for TLS & load-balancing.
- Harden your Tomcat (remove manager/host-manager in production or protect with strong passwords).

---

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

---

### Short interview-ready summary
"Create two EC2 instances (build + tomcat). On build node install JDK/Maven, clone repo and `mvn clean package`. Copy resulting WAR to tomcat node (SCP), place it in Tomcat's `webapps` and restart Tomcat or let it auto-deploy. Verify via logs and browser. Use systemd for Tomcat in production, and avoid committing keys to git."

---

**End of document.**

*You can copy this file into `README.md` in your GitHub repo or save as `docs/DEPLOYMENT_GUIDE.md`.*

