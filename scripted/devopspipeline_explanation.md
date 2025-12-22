pipeline {
    agent { ... }
    stages {
        stage('Checkout') { ... }
        stage('Build') { ... }
        stage('Deploy') { ... }
    }
}
