## Jenkins Installation Guide (Red Hat / CentOS / Rocky / AlmaLinux)

**Last Updated**: 07-Jan-2026
**Official Jenkins installation guide (Linux)**: https://www.jenkins.io/doc/book/installing/linux/

### Installation Steps

#### 1. Add Jenkins Repository

Downloads the official Jenkins YUM repository configuration file and saves it to `/etc/yum.repos.d/`, allowing the system to install and update Jenkins using `yum` or `dnf`.

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
```

#### 2. Upgrade System Packages

Updates all installed packages on the system to the latest available versions.

```bash
sudo yum upgrade -y
```

#### 3. Install Required Dependencies

Installs required dependencies for Jenkins.  
`fontconfig` is needed for font rendering, and **Java 21** is required to run Jenkins (Jenkins 2.426+ LTS requires Java 21).

**Recommended (Amazon Corretto Java 21 – official Amazon distribution, widely used on Amazon Linux)**:

```bash
sudo yum install fontconfig java-21-amazon-corretto -y
```

**Alternative (OpenJDK 21)**:

```bash
sudo yum install java-21-amazon-corretto* -y
```

#### Validation Commands

Verify Java installation:

```bash
java -version
# Expected output example:
# openjdk version "21.0.x" ...
# or Amazon Corretto 21...

rpm -qa | grep java-21
# Should list the installed Java package
```

#### Removal Command (If Needed)

To remove Amazon Corretto Java 21:

```bash
sudo yum remove java-21-amazon-corretto* -y
```

To remove OpenJDK 21:

```bash
sudo yum remove java-21-openjdk* -y
```

**Note**: After changing Java versions, restart Jenkins:
```bash
sudo systemctl restart jenkins
```

#### 4. Install Jenkins

Installs the Jenkins package from the configured Jenkins repository.

```bash
sudo yum install jenkins -y
```

#### 5. Reload systemd Configuration

Reloads systemd manager configuration to recognize the Jenkins service.

```bash
sudo systemctl daemon-reload
```

#### 6. Enable Jenkins Service

Enables the Jenkins service to start automatically on system boot.

```bash
sudo systemctl enable jenkins
```

#### 7. Start Jenkins Service

Starts the Jenkins service.

```bash
sudo systemctl start jenkins
```

#### 8. Check Jenkins Status

Displays the current status of the Jenkins service.

```bash
sudo systemctl status jenkins
```

### Unlock Jenkins & Initial Setup

#### Step 1: Open Jenkins in Browser

Access Jenkins using the Jenkins master server **public IP** and default port **8080**.

```
http://<PublicIP>:8080
```

#### Step 2: Unlock Jenkins

On the **Unlock Jenkins** screen, Jenkins asks for the **Administrator password**.

The password is stored on the Jenkins master server at:

```
/var/lib/jenkins/secrets/initialAdminPassword
```

Run the following command on the Jenkins master server to get the password:

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the output and paste it into the **Administrator password** field in the browser, then click **Continue**.

#### Step 3: Install Plugins

On the **Getting Started** screen:

* Select **Install suggested plugins**
* Jenkins installs the recommended plugins automatically

#### Step 4: Create First Admin User

Fill in the required details:

* Username
* Password
* Confirm password
* Full name
* Email address

*(Optional: You may choose **Skip and continue as admin**)*

Click **Save and Continue**.

#### Step 5: Finish Setup

* Jenkins completes the setup
* You are redirected to the Jenkins dashboard

### Important Jenkins Paths & Commands (Post-Installation)

After successful installation, here are the **key Jenkins paths** on Red Hat/CentOS/Rocky/AlmaLinux (default locations). Use these for troubleshooting, backups, and monitoring.

#### Key Jenkins Paths
| Path                                      | Purpose / Description                                                                 |
|-------------------------------------------|---------------------------------------------------------------------------------------|
| `/var/lib/jenkins`                        | **JENKINS_HOME** – Main directory (configs, jobs, plugins, users, secrets). Backup this! |
| `/var/lib/jenkins/secrets/initialAdminPassword` | Initial unlock password (single-use; deleted after setup)                            |
| `/var/lib/jenkins/workspace/<job-name>`   | Per-job workspace (source code, build artifacts during execution)                    |
| `/var/lib/jenkins/jobs/<job-name>`        | Job-specific configs, build history, and logs                                        |
| `/var/lib/jenkins/plugins`                | Installed plugins (.hpi/.jpi files)                                                   |
| `/etc/sysconfig/jenkins`                  | System configuration (JVM options, port, user – edit for customizations)             |
| `/var/log/jenkins/jenkins.log`            | Main Jenkins system log (errors, startup, plugin issues)                             |
| `/usr/share/jenkins/jenkins.war`          | Jenkins WAR file (core application – rarely modified)                                |

**Note**: To change JENKINS_HOME, edit `/etc/sysconfig/jenkins` and restart the service.

#### Checking Logs
Jenkins logs help debug startup issues, plugin errors, or build failures.

**View Main System Log** (most common for service issues):
```bash
sudo tail -f /var/log/jenkins/jenkins.log
# Follow live logs
# Or for full log:
sudo journalctl -u jenkins -f
```

**View Job-Specific Build Logs**:
- In Jenkins UI: Job → Build History → Console Output
- CLI: `java -jar jenkins-cli.jar -s http://localhost:8080 console <job-name> <build-number>`

**Full Log Rotation Check**:
```bash
sudo ls -la /var/log/jenkins/
# Shows log files and rotation
```

#### Checking if Port is Listening
Jenkins defaults to port **8080** (HTTP). Verify it's listening and accessible.

**Check if Port 8080 is Listening**:
```bash
# Using netstat (if installed)
sudo netstat -tlnp | grep 8080
# Expected: tcp6 0 0 :::8080 :::* LISTEN <pid>/java

# Using ss (modern alternative)
sudo ss -tlnp | grep 8080
# Expected: LISTEN 0 128 <ip>:8080 *:* users:(("java",pid=<pid>,fd=...))

# Using lsof
sudo lsof -i :8080
# Expected: java <pid> jenkins <ip>:8080 (LISTEN)
```

**Test Connectivity**:
```bash
# From local machine
curl -I http://localhost:8080
# Expected: HTTP/1.1 200 OK (or redirect to setup)

# From remote
curl -I http://<your-server-ip>:8080
# If fails: Check firewall/security group
```

**If Port Not Listening**:
- Restart: `sudo systemctl restart jenkins`
- Check status: `sudo systemctl status jenkins`
- Firewall: `sudo firewall-cmd --add-port=8080/tcp --permanent && sudo firewall-cmd --reload`

**Practice Tasks**:
- Tail the Jenkins log while starting a build → Observe real-time output.
- Use `ss` or `netstat` to confirm port 8080 is bound to Java process.
- If issues: Check `/var/log/jenkins/jenkins.log` for errors (e.g., Java version mismatch).

**References**:
- [Viewing Logs - Jenkins Docs](https://www.jenkins.io/doc/book/system-administration/viewing-logs/)
- [Systemd Services](https://www.jenkins.io/doc/book/system-administration/systemd-services/)