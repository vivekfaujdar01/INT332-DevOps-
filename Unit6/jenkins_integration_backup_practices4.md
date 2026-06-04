# Jenkins Advanced: GitHub Integration, Backup & Restore, and Pipeline Best Practices

This document covers advanced topics in Jenkins administration and pipeline management: integrating source control dynamically, setting up backup regimes, and writing efficient, secure, and production-ready CI/CD pipelines.

---

## 1. Jenkins and GitHub Integration

Connecting Jenkins with GitHub allows for automatic build triggers on code push, automated status reporting on Pull Requests (PRs), and secure code checkouts.

### A. Authentication Mechanisms
To clone repositories from GitHub, Jenkins requires authentication.
1.  **SSH Key Pair (Recommended for private repos)**:
    *   Generate a key pair on the Jenkins agent/controller: `ssh-keygen -t ed25519`.
    *   Add the **public key** (`.pub`) to GitHub: Repository -> Settings -> Deploy Keys (read-only) or GitHub Account Settings -> SSH Keys (write access).
    *   Add the **private key** to Jenkins: Manage Jenkins -> Credentials -> Global -> Add Credentials (select `SSH Username with private key`).
2.  **Personal Access Token (PAT)**:
    *   Generate a token in GitHub: Settings -> Developer Settings -> Personal Access Tokens (Classic or Fine-Grained) with `repo` permissions.
    *   Add the token to Jenkins as `Username with Password` or `Secret text`.

### B. Configuring Automatic Webhooks
A webhook notifies Jenkins immediately whenever a change is pushed to the repository.

1.  **In GitHub**:
    *   Go to Repository Settings -> **Webhooks** -> **Add webhook**.
    *   **Payload URL**: `http://<YOUR_JENKINS_SERVER_URL>/github-webhook/`
    *   **Content type**: `application/json`
    *   **Which events**: Choose `Just the push event` or select specific events (like Pull Requests).
2.  **In Jenkins (Pipeline/Freestyle configuration)**:
    *   Check the box: **GitHub trigger for GITScm polling** under Build Triggers.
    *   When code is pushed, GitHub sends a POST request to Jenkins, triggering the pipeline execution.

### C. Declaring Git Checkout in Pipelines
Using the standard `git` step or the more advanced `checkout` DSL:

```groovy
stage('Checkout Source') {
    steps {
        // Simple Git Checkout
        git branch: 'main', 
            credentialsId: 'github-ssh-creds', 
            url: 'git@github.com:username/my-repo.git'
    }
}
```

---

## 2. Backup & Restore Procedures

Jenkins stores all its configurations, jobs, users, and credentials as XML files inside the `$JENKINS_HOME` directory. Backing up this directory is essential for disaster recovery.

### A. Core Directory Structure of `$JENKINS_HOME`
Understanding what to back up helps manage storage sizes:
*   `config.xml`: Jenkins global configuration.
*   `jobs/`: Contains folders for each build job, including configuration and build histories. (Can exclude `builds/` if backup space is limited).
*   `plugins/`: Installed plugins. (Can be re-downloaded, but backing up saves setup time).
*   `secrets/`: **CRITICAL**. Contains keys used to encrypt credentials. Without this, your credential stores cannot be decrypted upon restore.
*   `users/`: User profiles and settings.
*   `workspace/`: **EXCLUDE**. Contains checkout files for builds; dynamically generated, do not back up.

### B. Backup Strategies

#### Method 1: ThinBackup Plugin (Recommended)
ThinBackup is a Jenkins plugin that automates directory backups:
1.  Install the **ThinBackup** plugin via the Plugin Manager.
2.  Go to **Manage Jenkins** -> **ThinBackup** -> **Settings**.
3.  Configure:
    *   **Backup directory**: Path on the server or mounted storage.
    *   **Schedule**: Cron expression (e.g., `0 2 * * *` for daily at 2 AM).
    *   **Options**: Check *Backup build results* (optional), *Clean up differential backups*, and *Move old backups*.
4.  Click **Backup Now** to verify operation.

#### Method 2: Shell Scripting (Automated Tar and Upload)
You can set up a Linux cron job to compress and upload the configurations to an external cloud storage provider (e.g., AWS S3):

```bash
#!/bin/bash
# backup_jenkins.sh
BACKUP_DIR="/backups/jenkins"
JENKINS_HOME="/var/lib/jenkins"
DATE=$(date +%F-%H%M)
BACKUP_FILE="${BACKUP_DIR}/jenkins-backup-${DATE}.tar.gz"

# Create backup directory
mkdir -p ${BACKUP_DIR}

# Tar config and secrets, exclude dynamic workspaces/caches
tar -czf ${BACKUP_FILE} \
  --exclude="${JENKINS_HOME}/workspace" \
  --exclude="${JENKINS_HOME}/caches" \
  --exclude="${JENKINS_HOME}/fingerprints" \
  ${JENKINS_HOME}

# Optional: Upload to AWS S3
# aws s3 cp ${BACKUP_FILE} s3://my-jenkins-backups-bucket/
```

### C. Restore Process
In the event of server failure:
1.  Provision a new machine and install the same version of Jenkins.
2.  Stop the Jenkins service: `sudo systemctl stop jenkins`.
3.  Extract the backup archive into the new `$JENKINS_HOME` directory:
    ```bash
    sudo tar -xzf jenkins-backup.tar.gz -C /var/lib/jenkins --strip-components=1
    ```
4.  Ensure ownership permissions are correct:
    ```bash
    sudo chown -R jenkins:jenkins /var/lib/jenkins
    ```
5.  Start the Jenkins service: `sudo systemctl start jenkins`.

---

## 3. Pipeline Best Practices

Writing maintainable, secure, and performant pipelines is key to a healthy CI/CD system.

### 1. Never Execute Builds on the Controller
The Controller should be dedicated to orchestration. Running heavy build steps on the Controller slows down the UI, exhausts memory, and presents a security threat since jobs can access Jenkins internal files.
*   **Best Practice**: Set **Number of executors** to `0` on the built-in controller node in Manage Jenkins -> Nodes.

### 2. Discard Old Builds (Log Rotation)
Retaining build history indefinitely will consume all available disk space over time.
*   **Best Practice**: Add the `buildDiscarder` option to your pipeline.

```groovy
pipeline {
    agent any
    options {
        // Keep only the last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages { ... }
}
```

### 3. Implement Build Timeouts
If a stage hangs (waiting for user input, network calls, or a test execution that got stuck), it can block the executor indefinitely.
*   **Best Practice**: Enforce timeouts on stages or globally.

```groovy
pipeline {
    agent any
    options {
        // Fail the pipeline if it runs longer than 2 hours
        timeout(time: 2, unit: 'HOURS')
    }
    stages { ... }
}
```

### 4. Parallel Stage Execution
Optimize build times by executing independent stages concurrently.
*   **Best Practice**: Use the `parallel` block.

```groovy
stage('Run Verification') {
    parallel {
        stage('Unit Tests') {
            steps { sh 'mvn test' }
        }
        stage('Static Code Analysis') {
            steps { sh 'mvn sonar:sonar' }
        }
        stage('Frontend Lint') {
            steps { sh 'npm run lint' }
        }
    }
}
```

### 5. Always Clean the Workspace
Always clean the workspaces before or after a build run to prevent disk pollution and source code cross-pollution from previous runs.
*   **Best Practice**: Use the `cleanWs()` step in the `post` block.

```groovy
pipeline {
    agent any
    stages { ... }
    post {
        always {
            cleanWs() // Deletes files inside workspace on the agent
        }
    }
}
```

### 6. Avoid Groovy Scripting in Declarative Pipelines
Declarative pipelines are designed to be simple config-like templates. Adding complex script blocks (`script { ... }`) defeats this purpose.
*   **Best Practice**: If you require complex logic, loops, or custom functions, write them in a **Jenkins Shared Library** or push the logic down into external shell/Python scripts.
