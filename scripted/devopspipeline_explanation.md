# Jenkins CI/CD Pipeline - Complete Guide

## 📋 Table of Contents
1. Pipeline Structure Breakdown
2. Each Stage Explained
3. Interview Variations (3 Different Pipelines)
4. Common Interview Questions

---

## 🏗️ PIPELINE STRUCTURE BREAKDOWN

### **1. Agent Section**
```groovy
agent {
  docker {
    image 'maven:3.8.4-openjdk-11'
    args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
  }
}
```

**What it does:**
- Runs pipeline inside a Docker container with Maven & Java 11
- `--user root` → Gives full permissions inside container
- `-v /var/run/docker.sock:/var/run/docker.sock` → Mounts Docker socket so container can build Docker images (Docker-in-Docker)

**Why needed:**
- Consistent build environment
- No need to install Maven/Java on Jenkins server

---

### **2. Environment Variables**
```groovy
environment {
  DOCKER_REGISTRY = 'docker.io'
  APP_NAME = 'ecommerce-api'
}
```

**What it does:**
- Defines variables accessible throughout pipeline
- Can be used in any stage as `${APP_NAME}`

---

## 🔄 EACH STAGE EXPLAINED

### **STAGE 1: Source Code Checkout**
```groovy
stage('Source Code Checkout') {
  steps {
    echo 'Cloning repository...'
    git branch: 'develop', url: 'https://github.com/mycompany/ecommerce-backend.git'
  }
}
```

**Purpose:** Downloads source code from GitHub

**Real scenario:** "First, Jenkins needs to get the latest code from our Git repository"

---

### **STAGE 2: Compile and Package**
```groovy
stage('Compile and Package') {
  steps {
    sh 'mvn clean install -DskipTests'
  }
}
```

**Purpose:** 
- `mvn clean` → Removes old build files
- `mvn install` → Compiles code and creates JAR/WAR file
- `-DskipTests` → Skips tests (we run them separately)

**Output:** Creates `.jar` file in `target/` folder

---

### **STAGE 3: Unit Tests**
```groovy
stage('Unit Tests') {
  steps {
    sh 'mvn test'
  }
  post {
    always {
      junit '**/target/surefire-reports/*.xml'
    }
  }
}
```

**Purpose:** 
- Runs JUnit tests
- `post always` → Publishes test results even if tests fail
- Jenkins shows test reports in UI

**Why separate from build:** 
- Can see exactly where failure occurred
- Test reports are generated

---

### **STAGE 4: Code Quality Check (SonarQube)**
```groovy
stage('Code Quality Check') {
  environment {
    SONAR_HOST = "http://52.45.123.89:9000"
  }
  steps {
    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
      sh '''
        mvn sonar:sonar \
          -Dsonar.projectKey=ecommerce-api \
          -Dsonar.host.url=${SONAR_HOST} \
          -Dsonar.login=${SONAR_TOKEN}
      '''
    }
  }
}
```

**Purpose:** 
- Analyzes code for bugs, vulnerabilities, code smells
- Sends results to SonarQube server

**Key points:**
- `withCredentials` → Securely injects SonarQube token
- Token stored in Jenkins credentials (not hardcoded)
- SonarQube dashboard shows code quality metrics

---

### **STAGE 5: Security Scan**
```groovy
stage('Security Scan') {
  steps {
    sh 'mvn dependency-check:check'
  }
}
```

**Purpose:** 
- Checks if dependencies have known vulnerabilities
- Uses OWASP Dependency Check plugin

**Example:** If using Log4j 2.14 (has vulnerability), it will flag it

---

### **STAGE 6: Docker Image Build**
```groovy
stage('Docker Image Build') {
  environment {
    IMAGE_TAG = "${APP_NAME}:${BUILD_NUMBER}"
    IMAGE_LATEST = "${APP_NAME}:latest"
  }
  steps {
    script {
      sh "docker build -t ${IMAGE_TAG} -t ${IMAGE_LATEST} ."
    }
  }
}
```

**Purpose:** 
- Creates Docker image from application
- Tags with build number (e.g., `ecommerce-api:45`)
- Also tags as `latest`

**Why two tags:**
- Build number → Specific version tracking
- Latest → For quick deployments/testing

---

### **STAGE 7: Push to Registry**
```groovy
stage('Push to Registry') {
  environment {
    DOCKER_IMAGE = "myusername/${APP_NAME}:${BUILD_NUMBER}"
  }
  steps {
    script {
      docker.withRegistry("https://${DOCKER_REGISTRY}", 'dockerhub-credentials') {
        sh "docker tag ${APP_NAME}:${BUILD_NUMBER} ${DOCKER_IMAGE}"
        sh "docker push ${DOCKER_IMAGE}"
      }
    }
  }
}
```

**Purpose:** 
- Uploads image to Docker Hub
- `withRegistry` → Logs into Docker Hub using credentials
- Now image is available for deployment

**Flow:**
1. Authenticate with Docker Hub
2. Tag image with username/repo format
3. Push to registry

---

### **STAGE 8: Update Kubernetes Manifests**
```groovy
stage('Update Kubernetes Manifests') {
  environment {
    REPO_NAME = "ecommerce-gitops"
    GITHUB_USER = "myusername"
  }
  steps {
    withCredentials([string(credentialsId: 'github-token', variable: 'GIT_TOKEN')]) {
      sh '''
        git config user.email "devops@mycompany.com"
        git config user.name "Jenkins CI"
        
        sed -i "s|image:.*|image: myusername/${APP_NAME}:${BUILD_NUMBER}|g" k8s-manifests/deployment.yaml
        
        git add k8s-manifests/deployment.yaml
        git commit -m "Deploy version ${BUILD_NUMBER} to production"
        git push https://${GIT_TOKEN}@github.com/${GITHUB_USER}/${REPO_NAME} HEAD:main
      '''
    }
  }
}
```

**Purpose:** GitOps approach
- Updates Kubernetes deployment file with new image tag
- Commits and pushes changes to Git
- ArgoCD (running separately) detects change and deploys

**Key command:**
- `sed -i` → Finds and replaces image tag in YAML file

**Before:**
```yaml
image: myusername/ecommerce-api:44
```

**After:**
```yaml
image: myusername/ecommerce-api:45
```

---

### **STAGE 9: Deploy to Staging**
```groovy
stage('Deploy to Staging') {
  steps {
    sh '''
      kubectl config use-context staging-cluster
      kubectl apply -f k8s-manifests/ -n staging
      kubectl rollout status deployment/${APP_NAME} -n staging
    '''
  }
}
```

**Purpose:** 
- Deploys directly to staging Kubernetes cluster
- Waits for deployment to complete
- `rollout status` → Confirms pods are running

---

### **POST ACTIONS**
```groovy
post {
  success {
    emailext (
      subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
      body: "Build succeeded",
      to: 'devops-team@mycompany.com'
    )
  }
  failure {
    emailext (
      subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
      body: "Build failed",
      to: 'devops-team@mycompany.com'
    )
  }
  always {
    cleanWs()
  }
}
```

**Purpose:**
- Sends email notifications
- `cleanWs()` → Cleans workspace after build (saves disk space)

---

## 🎯 THREE INTERVIEW VARIATIONS

### **VARIATION 1: Node.js Microservice**
```groovy
pipeline {
  agent {
    docker {
      image 'node:16-alpine'
      args '--user root'
    }
  }
  
  environment {
    APP_NAME = 'payment-service'
    REGISTRY = 'gcr.io/my-project'
  }
  
  stages {
    stage('Checkout') {
      steps {
        git branch: 'master', url: 'https://github.com/company/payment-api.git'
      }
    }
    
    stage('Install Dependencies') {
      steps {
        sh 'npm ci'
      }
    }
    
    stage('Run Tests') {
      steps {
        sh 'npm test'
      }
    }
    
    stage('Build Docker Image') {
      steps {
        sh 'docker build -t ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER} .'
      }
    }
    
    stage('Push to GCR') {
      steps {
        withCredentials([file(credentialsId: 'gcp-key', variable: 'GCP_KEY')]) {
          sh '''
            cat ${GCP_KEY} | docker login -u _json_key --password-stdin gcr.io
            docker push ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
          '''
        }
      }
    }
    
    stage('Update Helm Values') {
      steps {
        withCredentials([string(credentialsId: 'git-token', variable: 'TOKEN')]) {
          sh '''
            git clone https://${TOKEN}@github.com/company/helm-charts.git
            cd helm-charts
            sed -i "s/tag:.*/tag: ${BUILD_NUMBER}/" values.yaml
            git add values.yaml
            git commit -m "Update to version ${BUILD_NUMBER}"
            git push
          '''
        }
      }
    }
  }
}
```

---

### **VARIATION 2: Python Flask API**
```groovy
pipeline {
  agent {
    docker {
      image 'python:3.9-slim'
    }
  }
  
  environment {
    APP = 'user-service'
    ECR_REPO = '123456789.dkr.ecr.us-east-1.amazonaws.com/user-service'
  }
  
  stages {
    stage('Get Source') {
      steps {
        checkout scm
      }
    }
    
    stage('Setup Virtual Environment') {
      steps {
        sh '''
          python -m venv venv
          . venv/bin/activate
          pip install -r requirements.txt
        '''
      }
    }
    
    stage('Lint Code') {
      steps {
        sh '''
          . venv/bin/activate
          pylint app/
        '''
      }
    }
    
    stage('Unit Tests') {
      steps {
        sh '''
          . venv/bin/activate
          pytest tests/ --junitxml=report.xml
        '''
      }
      post {
        always {
          junit 'report.xml'
        }
      }
    }
    
    stage('Build Container') {
      steps {
        sh 'docker build -t ${APP}:${BUILD_NUMBER} .'
      }
    }
    
    stage('Push to ECR') {
      steps {
        script {
          docker.withRegistry("https://${ECR_REPO}", 'ecr:us-east-1:aws-creds') {
            sh 'docker tag ${APP}:${BUILD_NUMBER} ${ECR_REPO}:${BUILD_NUMBER}'
            sh 'docker push ${ECR_REPO}:${BUILD_NUMBER}'
          }
        }
      }
    }
    
    stage('Deploy via ArgoCD') {
      steps {
        sh '''
          argocd app set user-service --parameter image.tag=${BUILD_NUMBER}
          argocd app sync user-service
        '''
      }
    }
  }
}
```

---

### **VARIATION 3: Multi-Branch Spring Boot**
```groovy
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8-openjdk-17
    command: ['cat']
    tty: true
"""
    }
  }
  
  environment {
    SERVICE_NAME = 'order-processing'
    NAMESPACE = "${env.BRANCH_NAME == 'main' ? 'production' : 'development'}"
  }
  
  stages {
    stage('Checkout') {
      steps {
        git url: 'https://github.com/corp/order-service.git'
      }
    }
    
    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn clean package'
        }
      }
    }
    
    stage('SonarQube Analysis') {
      when {
        branch 'main'
      }
      steps {
        container('maven') {
          withSonarQubeEnv('sonar-server') {
            sh 'mvn sonar:sonar'
          }
        }
      }
    }
    
    stage('Quality Gate') {
      when {
        branch 'main'
      }
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
    
    stage('Build Image') {
      steps {
        sh 'docker build -t ${SERVICE_NAME}:${GIT_COMMIT} .'
      }
    }
    
    stage('Push Image') {
      steps {
        withDockerRegistry([credentialsId: 'harbor-creds', url: 'https://harbor.company.com']) {
          sh '''
            docker tag ${SERVICE_NAME}:${GIT_COMMIT} harbor.company.com/${SERVICE_NAME}:${GIT_COMMIT}
            docker push harbor.company.com/${SERVICE_NAME}:${GIT_COMMIT}
          '''
        }
      }
    }
    
    stage('Deploy') {
      steps {
        sh '''
          kubectl set image deployment/${SERVICE_NAME} \
            ${SERVICE_NAME}=harbor.company.com/${SERVICE_NAME}:${GIT_COMMIT} \
            -n ${NAMESPACE}
          
          kubectl rollout status deployment/${SERVICE_NAME} -n ${NAMESPACE}
        '''
      }
    }
  }
  
  post {
    success {
      slackSend (
        channel: '#deployments',
        color: 'good',
        message: "✅ ${SERVICE_NAME} deployed successfully to ${NAMESPACE}"
      )
    }
    failure {
      slackSend (
        channel: '#deployments',
        color: 'danger',
        message: "❌ ${SERVICE_NAME} deployment failed"
      )
    }
  }
}
```

---

## 🎤 COMMON INTERVIEW QUESTIONS

### Q1: "Why mount Docker socket?"
**Answer:** "We mount the Docker socket so the container running Jenkins can access the host's Docker daemon to build images. Without this, we couldn't build Docker images from inside a Docker container."

### Q2: "What's GitOps and why update manifests in Git?"
**Answer:** "GitOps means Git is the single source of truth. When we update deployment.yaml in Git with the new image tag, ArgoCD detects the change and automatically deploys it to Kubernetes. This gives us version control, audit trail, and easy rollbacks."

### Q3: "Why separate stages for build and test?"
**Answer:** "Separation helps identify exactly where failures occur. If build fails, we know it's a compilation issue. If tests fail, it's a logic issue. Also, we can generate test reports separately."

### Q4: "How do you handle secrets?"
**Answer:** "We use Jenkins Credentials Store. Tokens and passwords are stored securely in Jenkins and injected at runtime using `withCredentials()`. They're never hardcoded in the pipeline."

### Q5: "What if Docker push fails?"
**Answer:** "The pipeline will fail at that stage. We can add retry logic or notifications. The post section will send failure email to the team."

### Q6: "Difference between declarative and scripted pipeline?"
**Answer:** "Declarative (what we're using) has structured syntax with stages, easier to read. Scripted uses Groovy code, more flexible but complex. Declarative is recommended for most use cases."

### Q7: "How to deploy to production safely?"
**Answer:** "Add an input step for manual approval:
```groovy
stage('Deploy to Production') {
  steps {
    input message: 'Deploy to production?', ok: 'Deploy'
    sh 'kubectl apply -f manifests/ -n production'
  }
}
```

### Q8: "What's the purpose of BUILD_NUMBER?"
**Answer:** "Jenkins auto-increments BUILD_NUMBER for each run. We use it to tag Docker images uniquely (e.g., app:45, app:46). This allows version tracking and easy rollbacks."

---

## 💡 TIPS FOR INTERVIEW

1. **Start with overview:** "This is a CI/CD pipeline that builds, tests, and deploys a microservice"

2. **Explain flow naturally:** "First we checkout code, then build it, run tests, analyze quality, build Docker image, push to registry, and finally deploy to Kubernetes"

3. **Show understanding of tools:**
   - Jenkins for automation
   - Maven for building Java apps
   - SonarQube for code quality
   - Docker for containerization
   - Kubernetes for orchestration
   - ArgoCD for GitOps deployment

4. **Be ready to modify:** If they say "What if it's a Node.js app?", you can switch to npm commands

5. **Explain why, not just what:** Don't just say "this builds the app", say "this compiles source code into a deployable artifact"

---

## 🚀 HOW TO PRESENT IN INTERVIEW

**Opening:** "Let me walk you through a CI/CD pipeline I implemented for deploying microservices to Kubernetes using Jenkins, Docker, and GitOps practices."

**Then explain stage by stage, emphasizing:**
- Problem it solves
- Tools used
- Why that approach

**End with:** "This pipeline ensures every code commit goes through automated testing, security scanning, and follows GitOps principles for deployment, giving us full traceability and easy rollbacks."
