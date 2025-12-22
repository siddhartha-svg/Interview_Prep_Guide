'''
pipeline {
  agent {
    docker {
      image 'maven:3.8.4-openjdk-11'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
  
  environment {
    DOCKER_REGISTRY = 'docker.io'
    APP_NAME = 'ecommerce-api'
  }
  
  stages {
    stage('Source Code Checkout') {
      steps {
        echo 'Cloning repository...'
        git branch: 'develop', url: 'https://github.com/mycompany/ecommerce-backend.git'
      }
    }
    
    stage('Compile and Package') {
      steps {
        echo 'Building application...'
        sh 'mvn clean install -DskipTests'
      }
    }
    
    stage('Unit Tests') {
      steps {
        echo 'Running unit tests...'
        sh 'mvn test'
      }
      post {
        always {
          junit '**/target/surefire-reports/*.xml'
        }
      }
    }
    
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
    
    stage('Security Scan') {
      steps {
        echo 'Running security checks...'
        sh 'mvn dependency-check:check'
      }
    }
    
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
    
    stage('Deploy to Staging') {
      steps {
        echo 'Deploying to staging environment...'
        sh '''
          kubectl config use-context staging-cluster
          kubectl apply -f k8s-manifests/ -n staging
          kubectl rollout status deployment/${APP_NAME} -n staging
        '''
      }
    }
  }
  
  post {
    success {
      echo 'Pipeline completed successfully!'
      emailext (
        subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
        body: "Build succeeded. Check console output at ${env.BUILD_URL}",
        to: 'devops-team@mycompany.com'
      )
    }
    failure {
      echo 'Pipeline failed!'
      emailext (
        subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
        body: "Build failed. Check console output at ${env.BUILD_URL}",
        to: 'devops-team@mycompany.com'
      )
    }
    always {
      cleanWs()
    }
  }
}

'''
