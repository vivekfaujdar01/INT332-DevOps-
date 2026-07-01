# Jenkins and Maven Integration

Apache Maven is a popular build automation tool used primarily for Java projects. Integrating Maven with Jenkins allows you to compile code, run tests, package applications (JAR, WAR, EAR), and generate test/coverage reports as part of your CI/CD pipeline.

---

## 1. Maven Installation and Global Tool Configuration

For Jenkins to execute Maven commands, Maven must be installed and referenced within Jenkins' Global Tool Configuration.

### A. Prerequisites: Configuring the JDK
Since Maven requires Java to run, you must first configure a JDK installation:
1.  Go to **Manage Jenkins** -> **Tools** (in older versions, **Global Tool Configuration**).
2.  Scroll down to **JDK installations** and click **Add JDK**.
3.  Set the **Name** (e.g., `jdk-17`).
4.  Configure:
    *   **Install automatically**: Jenkins will download it from Oracle or Adoptium on demand.
    *   **Manual path**: Uncheck "Install automatically" and provide the directory path (e.g., `/usr/lib/jvm/java-17-openjdk-amd64` in `JAVA_HOME`).

### B. Configuring Maven in Jenkins Tools
1.  In the same **Tools** configuration screen, scroll to **Maven installations** and click **Add Maven**.
2.  Set the **Name** (e.g., `maven-3.9`).
3.  Choose one of two installation methods:
    *   **Method 1: Automatic Installation (Recommended)**: Check **Install automatically**, select the version from the dropdown menu (e.g., `3.9.6` from Apache), and Jenkins will download/unpack Maven on the agent during runtime.
    *   **Method 2: Manual Installation**: Uncheck **Install automatically** and provide the `MAVEN_HOME` directory path of a pre-installed Maven instance on the build machines.
4.  Click **Save**.

---

## 2. Running Maven Builds in Pipelines

You can run Maven inside your Jenkins pipelines using the Global Tool reference or via Docker containers.

### Method 1: Using Global Tool Configurations (Recommended for VM agents)
In a Declarative Pipeline, the `tools` directive automatically downloads and injects the configured Maven and JDK paths into the environmental `$PATH` for the pipeline.

```groovy
pipeline {
    agent any
    
    tools {
        maven 'maven-3.9' // Must match the name configured in Global Tool Configuration
        jdk 'jdk-17'      // Must match the name configured in Global Tool Configuration
    }
    
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/example/my-maven-app.git'
            }
        }
        stage('Compile') {
            steps {
                echo 'Compiling code...'
                sh 'mvn clean compile'
            }
        }
        stage('Package') {
            steps {
                echo 'Packaging JAR file...'
                // Skip unit tests to speed up compile (tests run in the next stage)
                sh 'mvn package -DskipTests'
            }
        }
    }
}
```

### Method 2: Running Maven via Docker Container Agents
If you use Dockerized agents, you don't need to configure Maven in the Global Tool Configuration. Instead, run the pipeline steps inside an official Maven Docker container.

```groovy
pipeline {
    agent {
        docker { 
            image 'maven:3.9-eclipse-temurin-17'
            // Mount maven repository cache to speed up subsequent builds
            args '-v /root/.m2:/root/.m2' 
        }
    }
    stages {
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

---

## 3. Test Reports (JUnit)

When Maven runs tests (using the `maven-surefire-plugin`), it creates XML test reports in the `target/surefire-reports/` folder. Jenkins can parse these reports and display test results (passed, failed, skipped, and historical trends) directly on the build dashboard.

### Recording Test Results in the Pipeline
Use the `junit` step, which is best placed in a `post { always { ... } }` block to ensure tests are processed even if some of them fail.

```groovy
pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }
    stages {
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
    post {
        always {
            // Jenkins parses Surefire reports and displays a Test Result Graph
            junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
        }
    }
}
```

---

## 4. Code Coverage (JaCoCo)

Code coverage measures what percentage of your source code is executed during automated tests. **JaCoCo** (Java Code Coverage) is the industry-standard tool for Java projects.

### A. Configuring `pom.xml` for JaCoCo
To generate coverage data, add the `jacoco-maven-plugin` to the `build` section of your Maven `pom.xml`:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.11</version>
            <executions>
                <!-- Prepares the agent property before tests run -->
                <execution>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <!-- Generates report in target/site/jacoco/ after tests -->
                <execution>
                    <id>report</id>
                    <phase>test</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### B. Recording JaCoCo Coverage in Jenkins
1.  Install the **JaCoCo Plugin** in Jenkins via the Plugin Manager.
2.  Use the `jacoco` step in the `post` block of your pipeline. This parses the coverage file (`target/jacoco.exec`) and creates coverage charts.

```groovy
pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }
    stages {
        stage('Test & Analyze') {
            steps {
                // Generates test results and jacoco.exec
                sh 'mvn clean test' 
            }
        }
    }
    post {
        always {
            junit '**/target/surefire-reports/*.xml'
            
            // Record JaCoCo coverage metrics
            jacoco(
                execPattern: '**/target/jacoco.exec',
                classPattern: '**/target/classes',
                sourcePattern: '**/src/main/java',
                exclusionPattern: '**/*Test.class'
            )
        }
    }
}
```

### Jenkins Coverage Dashboard
Once configured, the Jenkins build page will feature:
*   A **Coverage Trend Graph** showing historical changes in code coverage.
*   A detailed report showing coverage percentages broken down by:
    *   **Instructions** (Bytecode level)
    *   **Branches** (Decision points, e.g., `if/else`)
    *   **Complexity** (Cyclomatic complexity)
    *   **Lines**
    *   **Methods**
    *   **Classes**
