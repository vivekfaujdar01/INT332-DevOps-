# Jenkins CI/CD Deployment Flows: Triggers, Shared Libraries, Agents & Deployments

This document outlines the mechanisms Jenkins uses to trigger builds, reuse code via shared libraries, distribute workloads across various agent types, and execute deployments to remote servers and cloud environments.

---

## 1. Triggering Builds (pollSCM vs. Webhook)

Automating pipelines requires triggering them when code changes. The two primary methods are **pollSCM** (polling) and **Webhooks** (push notifications).

| Feature | pollSCM (Pull) | Webhooks (Push - Recommended) |
| :--- | :--- | :--- |
| **Direction** | Jenkins queries the SCM server. | SCM server pushes event to Jenkins. |
| **Latency** | Delayed (depends on polling interval). | Instant (starts build seconds after commit). |
| **Server Load** | High (constant requests to SCM provider). | Very Low (only sends requests on actual events). |
| **Network Req.** | Jenkins can remain private / behind firewall. | Jenkins must be accessible to SCM (or use a proxy). |

### A. pollSCM Configuration
Under the `triggers` block of a declarative pipeline, you define pollSCM using standard cron-like syntax.
```groovy
pipeline {
    agent any
    triggers {
        // Poll SCM every 15 minutes
        pollSCM('H/15 * * * *')
    }
    stages { ... }
}
```
*Note: `H` (Hash) is used in Jenkins to distribute the load across the system, preventing all jobs from polling at the exact same minute.*

### B. Webhook Trigger Configuration
When using webhooks, the pipeline uses a trigger definition (like `githubPush()`) and listens for incoming POST payloads.
```groovy
pipeline {
    agent any
    triggers {
        // Triggers the build when Git webhook notifies a push
        githubPush()
    }
    stages { ... }
}
```

---

## 2. Pipeline Shared Libraries

As teams create more pipelines, they often repeat code (e.g., standard Slack notification code, docker build wrappers). **Pipeline Shared Libraries** allow you to write reusable Groovy code and share it across multiple repositories.

### A. Directory Structure of a Shared Library
A shared library must be placed in a dedicated Git repository with this exact structure:

```text
my-shared-library/
├── src/                     # Standard Groovy source files (Object-Oriented classes)
│   └── org
│       └── company
│           └── Helper.groovy
├── vars/                    # Global variables and custom steps (called directly in Jenkinsfile)
│   ├── buildApp.groovy
│   └── sendSlackAlert.groovy
└── resources/               # Static resources (scripts, JSON, YAML) loaded at runtime
    └── config.json
```

*   **`vars/sendSlackAlert.groovy` Example:**
    ```groovy
    // vars/sendSlackAlert.groovy
    def call(String status) {
        slackSend channel: '#ci-alerts',
                  color: (status == 'SUCCESS' ? '#00FF00' : '#FF0000'),
                  message: "Build ${env.JOB_NAME} - Run #${env.BUILD_NUMBER} ended with status: ${status}"
    }
    ```

### B. Configuring the Library in Jenkins
1.  Go to **Manage Jenkins** -> **System** (or **Configure System**).
2.  Scroll to **Global Pipeline Libraries** and click **Add**.
3.  Set:
    *   **Name**: `my-shared-library`
    *   **Default version**: `main` (or a branch/tag name).
    *   **Retrieval Method**: Modern SCM -> Git (specify the repository URL and credentials).
4.  Click **Save**.

### C. Using the Shared Library in a Jenkinsfile
Use the `@Library` annotation at the top of the `Jenkinsfile`. The trailing underscore (`_`) is required to import all global steps from the `vars/` directory.

```groovy
@Library('my-shared-library@main') _

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Running build stage...'
            }
        }
    }
    post {
        always {
            // Invokes the custom step sendSlackAlert.groovy
            sendSlackAlert(currentBuild.currentResult)
        }
    }
}
```

---

## 3. Jenkins Agents (SSH, SFTP, and Container-based)

To run distributed builds, the Controller needs to execute commands on remote servers (Agents).

### A. SSH / SFTP-Based Agents
The Controller uses SSH to connect to a target Linux server, copies the `agent.jar` file over, and runs it.
1.  **Connection**: The Jenkins controller acts as an SSH client.
2.  **Configuration**:
    *   In **Manage Jenkins** -> **Nodes** -> **New Node**.
    *   **Launch Method**: Select `Launch agents via SSH`.
    *   **Host**: Agent IP/Domain.
    *   **Credentials**: Add SSH private key credentials (corresponding to the public key in the agent's `~/.ssh/authorized_keys` file).
    *   **Host Key Verification**: Select verification strategy (e.g., `Known hosts file` or `Manually trusted key`).
3.  Upon launch, Jenkins connects, automatically transfers `agent.jar` using SFTP (or SCP), and runs:
    `java -jar agent.jar`

### B. Container-Based Agents (Docker & Kubernetes)
Instead of dedicating VM servers as agents, modern installations provision temporary, lightweight containers.

*   **Docker Engine Agent**:
    Using the Docker plugin, the Controller communicates with a remote Docker daemon API. When a build is scheduled, the Controller launches a container using a pre-configured image (e.g., `jenkins/inbound-agent`) with the necessary tools, executes the build, and destroys the container afterwards.
*   **Kubernetes Pod Agent**:
    Using the Kubernetes plugin, Jenkins spins up a Kubernetes Pod containing one or more containers (e.g., a Maven container, a Node container, and a JNLP agent container) in a cluster to run a pipeline.
    ```groovy
    pipeline {
        agent {
            kubernetes {
                yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.9-eclipse-temurin-17
                    command: ['cat']
                    tty: true
                '''
            }
        }
        stages {
            stage('Run in Pod') {
                steps {
                    container('maven') {
                        sh 'mvn --version' // Runs inside the Maven container of the pod
                    }
                }
            }
        }
    }
    ```

---

## 4. Deployments to Servers and Clouds

Once artifacts are built and tested, Jenkins can deploy them to target environments.

### A. Deployment to VM Servers via SSH
For deployment to legacy VM servers, use the `sshagent` step to securely inject keys and deploy via `scp`/`rsync`.

```groovy
stage('Deploy to Linux VM') {
    steps {
        // Injects SSH key from credentials store into ssh-agent
        sshagent(['webserver-ssh-key-id']) {
            // Copy artifact to remote web server
            sh 'scp target/app.war ubuntu@webserver-ip:/opt/tomcat/webapps/'
            // Restart target service
            sh "ssh ubuntu@webserver-ip 'sudo systemctl restart tomcat'"
        }
    }
}
```

### B. Deployment to Public Clouds (AWS)
Deploying to AWS requires injecting credentials using AWS plugins or standard environment variables.

```groovy
stage('Deploy to AWS S3') {
    steps {
        // Inject AWS access credentials
        withCredentials([usernamePassword(credentialsId: 'aws-deploy-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
            // Upload compiled artifact to AWS S3 bucket
            sh 'aws s3 cp target/app.jar s3://my-production-bucket-s3/app-latest.jar'
        }
    }
}
```

### C. Deployment to Kubernetes Cluster
To deploy containerized applications to Kubernetes, use Kubeconfig files to securely authenticate `kubectl`.

```groovy
stage('Deploy to Kubernetes') {
    steps {
        // Binds the Kubeconfig file credential securely
        withKubeConfig([credentialsId: 'k8s-kubeconfig-id']) {
            // Update deployment image in the cluster
            sh 'kubectl set image deployment/myapp-deployment myapp=ghcr.io/username/myapp:${BUILD_NUMBER}'
            sh 'kubectl rollout status deployment/myapp-deployment'
        }
    }
}
```
