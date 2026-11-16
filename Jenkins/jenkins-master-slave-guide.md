## Jenkins Distributed Setup: Connect Master to 2 Permanent Agents (Slaves)

**Goal**: Configure **two permanent agents** (e.g., `agent-1` and `agent-2`) to offload builds from the Jenkins controller (master)..
**Connection Method**: SSH (password-less) – most secure and reliable.  
**Assumptions**:
- All machines are Amazon Linux 2 (common for EC2).
- Jenkins controller is already installed and running.
- You have SSH access to all three machines (controller + 2 agents).

### Phase 1: Prepare the Two Agent Machines

Run these steps **on each agent** (`agent-1` and `agent-2` separately).

```bash
# 1. Login to the agent machine (as ec2-user or your SSH user)
ssh -i "<path-to-your-key.pem>" ec2-user@<PublicIP-or-DNS>

# 2. Switch to root
sudo su -

# 3. Install Required Dependencies
# fontconfig is needed for font rendering, and Java 21 is required to run Jenkins (Jenkins 2.426+ LTS requires Java 21).

# Recommended (Amazon Corretto Java 21 – official Amazon distribution, widely used on Amazon Linux):
sudo yum install fontconfig java-21-amazon-corretto -y

# Alternative (OpenJDK 21):
sudo yum install fontconfig java-21-openjdk -y

# Validation Commands
java -version
# Expected output example:
# openjdk version "21.0.x" ...
# or Amazon Corretto 21...

rpm -qa | grep java-21
# Should list the installed Java package

# 4. Create jenkins user (Jenkins agent will run as this user)
useradd -m -s /bin/bash jenkins

# 5. Create .ssh directory for jenkins user
mkdir -p /home/jenkins/.ssh
chown jenkins:jenkins /home/jenkins/.ssh
chmod 700 /home/jenkins/.ssh
```

**Repeat** the above for the second agent (`agent-2`).

### Phase 2: Prepare the Jenkins Controller (Master) – Detailed Step-by-Step

Run these steps **once** on the Jenkins controller machine.

```bash
# 1. Login to the controller machine and switch to root
ssh -i "<your-key.pem>" ec2-user@<controller-public-ip-or-dns>
sudo su -

# 2. Ensure the 'jenkins' user exists
# The Jenkins RPM package normally creates this user automatically.
# This command checks and creates it only if missing.
id -u jenkins &>/dev/null || useradd -m -s /bin/bash -d /var/lib/jenkins jenkins
passwd jenkins
sudo usermod -s /bin/bash jenkins
# 3. Create .ssh directory for the jenkins user (with correct permissions)
mkdir -p /var/lib/jenkins/.ssh
chown jenkins:jenkins /var/lib/jenkins/.ssh
chmod 700 /var/lib/jenkins/.ssh
```

#### 4. Generate SSH Key Pair on Controller (as jenkins user)

```bash
# Switch to jenkins user
su - jenkins

# Navigate to .ssh directory
cd ~/.ssh || mkdir ~/.ssh && cd ~/.ssh

# Generate RSA key pair (4096 bits, no passphrase)
ssh-keygen -t rsa -b 4096 -f id_rsa -q -N ""

# Set correct permissions
chmod 600 id_rsa
chmod 644 id_rsa.pub

# Display the public key (you will need it for agents)
cat /var/lib/jenkins/.ssh/id_rsa.pub

# Exit back to root
exit
```

> **Result**:  
> Private key: `/var/lib/jenkins/.ssh/id_rsa`  
> Public key: `/var/lib/jenkins/.ssh/id_rsa.pub`

#### 5. Copy the Public Key to Both Agents (Manual – Simplest & Most Reliable on EC2)

**No ssh-copy-id needed** — manual copy avoids password/shell problems.

**On the master** (as jenkins user):
```bash
sudo -u jenkins
# Copy public key to clipboard (or note it)
cat ~/.ssh/id_rsa.pub
exit
```

**On each agent** (login as ec2-user with PEM key):
```bash
ssh -i "<your-key.pem>" ec2-user@<agent-ip>

# As ec2-user on agent:
sudo su -

# Add the master's public key (paste the key you copied!)
echo "ssh-rsa AAAAB3NzaC1yc2E... jenkins@master" >> /home/jenkins/.ssh/authorized_keys
chown jenkins:jenkins /home/jenkins/.ssh/authorized_keys
chmod 600 /home/jenkins/.ssh/authorized_keys

exit
```

**Repeat** for the second agent.

#### 6. Add Agents' Host Keys to known_hosts (Avoid Host Key Verification Prompts)

```bash
# Run as root
ssh-keyscan -H <agent-1-ip-or-hostname> >> /var/lib/jenkins/.ssh/known_hosts
ssh-keyscan -H <agent-2-ip-or-hostname> >> /var/lib/jenkins/.ssh/known_hosts

# Fix ownership and permissions
chown jenkins:jenkins /var/lib/jenkins/.ssh/known_hosts
chmod 600 /var/lib/jenkins/.ssh/known_hosts
```

#### 7. Test Password-less SSH Connectivity (From Master to Slave Machines)

```bash
# Test connection to both agents as the jenkins user
sudo -u jenkins
ssh jenkins@<agent-1-ip>
ssh jenkins@<agent-2-ip>
# Expected output: the hostname of the agent (no password prompt, no host key warning)
exit
```

**Success Criteria**:
- No password asked
- No "Are you sure you want to continue connecting?" prompt
- Returns the agent’s hostname immediately

If any test fails:
- Double-check `authorized_keys` on agent has correct public key
- Verify file permissions (`700` for `.ssh`, `600` for files)
- Check firewall/security group allows SSH (port 22) from controller to agents

Once both tests pass → proceed to **Phase 3** (adding agents in Jenkins UI).

This detailed Phase 2 ensures reliable, secure, password-less SSH from controller to agents — exactly what Jenkins needs for "Launch agents via SSH".

### Phase 3: Add Agents in Jenkins Web UI

1. Open Jenkins in browser: `http://<controller-public-ip>:8080`
2. Go to **Manage Jenkins → Nodes and Clouds → New Node**
3. Repeat the following for **each agent**:

#### For Agent-1
- **Node name**: `agent-1` (or descriptive name)
- **Type**: Permanent Agent
- Click **Create**

**Configuration**:
- **# of executors**: 2–4 (depending on CPU)
- **Remote FS root**: `/home/jenkins` (or where you created jenkins user)
- **Labels**: `linux agent-1 docker` (use meaningful labels)
- **Usage**: "Use this node as much as possible"
- **Launch method**: **Launch agents via SSH**
  - **Host**: Private or public IP of agent-1 (use private if in same VPC)
  - **Credentials**: Click **Add** → **Jenkins**
    - **Kind**: SSH Username with private key
    - **Scope**: Global
    - **ID**: e.g., `agent-ssh-key`
    - **Username**: `jenkins` (the user we created on agent)
    - **Private Key**: From file on server → `/var/lib/jenkins/.ssh/id_rsa` (this is where the key lives on the jenkins master server).
  - Select the credential
- **Host Key Verification Strategy**: "Known hosts file" (recommended)
- Click **Save**

#### For Agent-2
Repeat with adjusted name, labels, host.

### Verification

1. Manage Nodes → Agents should show **Online** (green).
2. Test pipeline with label 'linux' → Runs on an agent.

**Additional Commands**:
- Logs: `sudo tail -f /var/log/jenkins/jenkins.log`
- Port Check: `sudo netstat -tlnp | grep 8080` or `sudo ss -tlnp | grep 8080`

**Done!** Your Jenkins controller now distributes builds across two permanent agents.

**References**:
- https://www.jenkins.io/doc/book/using/using-agents/
