---

# 🔷 Jenkins Declarative Pipeline – End-to-End Explanation

---

## 1️⃣ Pipeline & Agent

```groovy
pipeline {
    agent any
```

### What it means

* This is a **Declarative Jenkins pipeline**
* `agent any` → Jenkins can run this job on **any available executor/agent**
* No dependency on a specific node (master/agent/docker)

### Interview point

> “We keep the agent generic so the pipeline is portable across environments.”

---

## 2️⃣ Environment Variables

```groovy
environment {
    DOCKER_IMAGE = "myrepo/myapp:${BUILD_NUMBER}"
    SONAR_HOST = "http://sonarqube:9000"
}
```

### What happens

* `BUILD_NUMBER` → Jenkins auto-generated unique build ID
* Docker image gets **versioned automatically**
* SonarQube URL centralized in one place

### Why this is important

* No hard-coding inside stages
* Easy to change image naming or Sonar URL

### Interview line

> “Environment block helps in reusability and consistency across stages.”

---

## 3️⃣ Checkout Code Stage

```groovy
stage('Checkout Code') {
    steps {
        git branch: 'main',
            url: 'https://github.com/org/sample-app.git'
    }
}
```

### What happens

* Jenkins pulls source code from GitHub
* Always checks out the **main branch**
* Workspace gets populated with application code

### Why first stage

* CI always starts with **source control**
* All later stages depend on this code

### Interview line

> “Checkout is always the first step in CI to ensure we build the latest code.”

---

## 4️⃣ Build & Test Stage

```groovy
stage('Build & Test') {
    steps {
        sh '''
          mvn clean test
          mvn package
        '''
    }
}
```

### What happens internally

1. `mvn clean` → removes old build artifacts
2. `mvn test` → runs unit tests
3. `mvn package` → creates JAR/WAR file

### Why both build & test together

* Tests validate code **before deployment**
* Fails fast if application is broken

### Interview line

> “We ensure code quality by running tests before moving to analysis or deployment.”

---

## 5️⃣ SonarQube Analysis (Static Code Analysis)

```groovy
stage('SonarQube Analysis') {
    steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
            sh """
              mvn sonar:sonar \
              -Dsonar.host.url=${SONAR_HOST} \
              -Dsonar.login=${SONAR_TOKEN}
            """
        }
    }
}
```

### What happens

* SonarQube scans code for:

  * Bugs
  * Code smells
  * Vulnerabilities
  * Coverage issues
* Authentication done using **Jenkins credentials**

### Security best practice

* Token is **never hard-coded**
* Stored securely in Jenkins credentials store

### Interview line

> “Static analysis helps us catch security and quality issues early in the pipeline.”

---

## 6️⃣ Build Docker Image

```groovy
stage('Build Docker Image') {
    steps {
        sh 'docker build -t ${DOCKER_IMAGE} .'
    }
}
```

### What happens

* Docker image built using Dockerfile
* Image tagged with Jenkins `BUILD_NUMBER`
* Every build gets a **unique image version**

### Why this is important

* Ensures traceability
* Easy rollback using older tags

### Interview line

> “Dockerizing the app ensures consistency across environments.”

---

## 7️⃣ Push Docker Image

```groovy
stage('Push Docker Image') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            sh '''
              echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
              docker push ${DOCKER_IMAGE}
            '''
        }
    }
}
```

### What happens

* Jenkins logs into Docker registry securely
* Pushes image to Docker Hub (or private registry)

### Why credentials block

* Avoids exposing passwords
* Centralized credential management

### Interview line

> “We use Jenkins credentials to securely authenticate with Docker registry.”

---

## 8️⃣ Deploy to Kubernetes

```groovy
stage('Deploy to Kubernetes') {
    steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
            sh '''
              sed -i "s|IMAGE_TAG|${BUILD_NUMBER}|g" k8s/deployment.yaml
              kubectl apply -f k8s/deployment.yaml
              kubectl rollout status deployment/myapp
            '''
        }
    }
}
```

### What happens step by step

1. Jenkins loads **kubeconfig** securely
2. Updates image tag in deployment manifest
3. Applies Kubernetes deployment
4. Waits for rollout completion

### Why rollout status

* Confirms deployment success
* Prevents silent failures

### Interview line

> “This ensures zero-downtime deployment and validates rollout success.”

---

## 9️⃣ Post Actions

```groovy
post {
    success {
        echo "CI/CD Pipeline completed successfully"
    }
    failure {
        echo "Pipeline failed"
    }
}
```

### What happens

* Runs **after pipeline execution**
* Separate handling for success & failure
* Useful for notifications (Slack, email)

### Interview line

> “Post block helps in monitoring and alerting.”

---

# 🔥 END-TO-END FLOW (One-Line Summary)

> Code → Build → Test → Analyze → Dockerize → Push → Deploy → Verify

---

## ⭐ Interviewer Impression

If you explain like this:

* Logical flow
* Security aware
* Production mindset
* Clear CI/CD understanding

👉 **They will rate you GOOD / STRONG**

---

If you want next:

* **2-minute interview explanation**
* **Common failure scenarios**
* **How to improve this for production**

Just tell me 👍
