pipeline {
  agent any

  options {
    timestamps()
    skipDefaultCheckout(true)
  }

  environment {
    IMAGE_NAME = "spring-petclinic"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Test') {
      steps {
        sh '''
          chmod +x mvnw
          ./mvnw -B clean package
        '''
      }
      post {
        always { junit '**/target/surefire-reports/*.xml' }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          docker version
          docker build -t ${IMAGE_NAME}:${BRANCH_NAME}-${BUILD_NUMBER} .
        '''
      }
    }

    stage('Smoke Test Container') {
      steps {
        sh '''
          docker rm -f petclinic-smoke || true
          docker run -d --name petclinic-smoke -p 18080:8080 ${IMAGE_NAME}:${BRANCH_NAME}-${BUILD_NUMBER}
          sleep 10
          curl -f http://localhost:18080/ || (docker logs petclinic-smoke && exit 1)
          docker rm -f petclinic-smoke
        '''
      }
    }

    stage('Archive Artifact') {
      steps {
        archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
      }
    }
  }
}

