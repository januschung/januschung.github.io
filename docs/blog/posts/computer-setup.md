---
date: 2025-05-15
categories:
  - aws
  - kubernetes
  - devops
---
# Bootstrapping My Linux Desktop and MacBook for Dev Work

After transitioning through two new jobs recently, I had the opportunity (and challenge) to set up fresh dev environments on both a Linux desktop and a MacBook. Here’s my comprehensive checklist and setup notes, including tooling, config files, and references I found useful.

![Set up new computer for dev work](../../assets/blog/computer-setup/banner.jpg)

<!-- more -->

## Software Setup

### 1. Docker Desktop
- **Mac**:  
  [Install Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac/)
- **Linux**:
  ```bash
  sudo apt-get update
  sudo apt-get install docker.io
  sudo systemctl enable --now docker
  sudo usermod -aG docker $USER
  ```

### 2. Visual Studio Code
- **Mac/Linux**:  
  Download from [https://code.visualstudio.com/](https://code.visualstudio.com/)

### 3. IntelliJ IDEA
- **Mac/Linux**:  
  Download from [https://www.jetbrains.com/idea/download/](https://www.jetbrains.com/idea/download/)  
  Or use **JetBrains Toolbox**.

### 4. Slack
- **Mac**:  
  Download from [https://slack.com/downloads/mac](https://slack.com/downloads/mac)
- **Linux**:
  ```bash
  sudo snap install slack --classic
  ```

### 5. GitHub Copilot
- Install the GitHub Copilot plugin in:
  - [VS Code](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot)
  - [IntelliJ](https://plugins.jetbrains.com/plugin/17718-github-copilot)

### 6. Homebrew for Mac
Install only if you are using it for `terraform`, `npm`, or `jq`.
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 7. Kubectl
- **Mac**:
  ```bash
  curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
  chmod +x kubectl
  sudo mv kubectl /usr/local/bin/
  ```
- **Linux**:
  ```bash
  curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  chmod +x kubectl
  sudo mv kubectl /usr/local/bin/
  ```

### 8. Java (OpenJDK)
- **Mac**:  
  Download OpenJDK DMG from [https://jdk.java.net/](https://jdk.java.net/)
- **Linux**:
  ```bash
  sudo apt install openjdk-17-jdk
  ```

### 9. Python
- **Mac**:  
  Download official installer from [https://www.python.org/downloads/mac-osx/](https://www.python.org/downloads/mac-osx/)
- **Linux**:
  ```bash
  sudo apt install python3 python3-pip
  ```

### 10. Terraform
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```
[Terraform Install Guide](https://developer.hashicorp.com/terraform/downloads)

### 11. npm (via Node.js)
```bash
brew install node
```
[Node.js Docs](https://nodejs.org/)

### 12. jq
- **Mac**:
  ```bash
  brew install jq
  ```
- **Linux**:
  ```bash
  sudo apt install jq
  ```

---

## Configuration Files

### 1. AWS Config

#### Set Up AWS CLI

**Mac**
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

**Linux**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
[Install Docs](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

Set up AWS config and credential

```bash
aws configure
```

follow the prompt:
```bash
AWS Access Key ID [None]: YOUR_ACCESS_KEY_ID
AWS Secret Access Key [None]: YOUR_SECRET_ACCESS_KEY
Default region name [None]: us-east-1
Default output format [None]: json
```

### 2. Kubeconfig
Copy your kubeconfig to:
```bash
~/.kube/config
```
Or use:
```bash
export KUBECONFIG=/path/to/kubeconfig.yaml
```

### 3. GitHub SSH
```bash
ssh-keygen -t ed25519 -C "you@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

Then add the public key to your GitHub account:
```bash
cat ~/.ssh/id_ed25519.pub
```
