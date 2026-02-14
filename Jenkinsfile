pipeline {

    agent { label 'docker' }

    options {
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        IMAGE_NAME = "spring-petclinic"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Start Postgres (Compose)') {
            steps {
                sh '''
                  docker compose -p "petclinic-${BUILD_NUMBER}" up -d
                '''
            }
        }

        stage('Build & Test') {
            steps {
                sh '''
                  chmod +x mvnw;
                  ./mvnw clean install -DskipTests
                '''
            }
            // post {
            //     always {
            //         junit '**/target/surefire-reports/*.xml'
            //     }
            // }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
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

    post {
        always {
            sh '''
              docker compose -p "petclinic-${BUILD_NUMBER}" down -v || true
            '''
        }
    }
}

