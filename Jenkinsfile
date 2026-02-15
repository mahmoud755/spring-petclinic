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

        stage('Build & Test (Skip PostgresIntegrationTests)') {
            steps {
                sh '''
                  set -euxo pipefail
                  chmod +x mvnw
                  ./mvnw -B clean package -Dtest=!PostgresIntegrationTests
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
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
          set -euxo pipefail
          chmod +x mvnw
          ./mvnw -B clean test -Dtest=!PostgresIntegrationTests
        '''
      }
      post {
        always { junit '**/target/surefire-reports/*.xml' }
      }
    }
            docker rm -f petclinic-smoke || true

            docker run -d \
              --name petclinic-smoke \
              --network jenkins \
              ${IMAGE_NAME}:${BRANCH_NAME}-${BUILD_NUMBER}

            echo "Waiting for app..."
            sleep 15

            curl -f http://petclinic-smoke:8080/ \
              || (docker logs petclinic-smoke && exit 1)

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

    // post {
    //     always {
    //         sh '''
    //           docker compose -p "petclinic-${BUILD_NUMBER}" down -v || true
    //         '''
    //     }
    // }
}
