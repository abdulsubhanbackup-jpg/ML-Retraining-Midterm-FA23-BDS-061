
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE = "student-ml-midterm"
        DOCKER_IMAGE = "abdulsubhan480/ml-retraining-midterm"
        CONTAINER = "student-ml-api"
        PORT = "8000"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate and Train Model') {
            steps {
                sh '''
                    set -e
                    test -f config.json
                    test -f dataset/train.csv
                    docker build \
                        --build-arg GIT_COMMIT=${GIT_COMMIT} \
                        -t ${IMAGE}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Test Candidate') {
            steps {
                sh '''
                    set -e
                    docker rm -f student-ml-candidate || true

                    docker run -d \
                        --name student-ml-candidate \
                        -p 8001:8000 \
                        ${IMAGE}:${BUILD_NUMBER}

                    for i in $(seq 1 30); do
                        if curl -fsS http://localhost:8001/metrics > candidate-metrics.json; then
                            break
                        fi
                        sleep 2
                    done

                    test -s candidate-metrics.json

                    python3 -c "
import json
m = json.load(open('candidate-metrics.json'))
assert m['student_id'] == 'FA23-BDS-061'
assert m['training_rows'] >= 100
print(m)
"

                    docker rm -f student-ml-candidate
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        set -e

                        echo "$DOCKER_PASS" | docker login \
                            -u "$DOCKER_USER" --password-stdin

                        docker tag ${IMAGE}:${BUILD_NUMBER} \
                            ${DOCKER_IMAGE}:${BUILD_NUMBER}

                        docker tag ${IMAGE}:${BUILD_NUMBER} \
                            ${DOCKER_IMAGE}:latest

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -e

                    docker rm -f ${CONTAINER} || true

                    docker run -d \
                        --restart unless-stopped \
                        --name ${CONTAINER} \
                        -p ${PORT}:8000 \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER}

                    sleep 5

                    curl -fsS http://localhost:${PORT}/metrics
                '''
            }
        }
    }

    post {
        success {
            emailext(
                subject: "SUCCESS: ${JOB_NAME} #${BUILD_NUMBER}",
                body: """
Hello,

Your Git commit has been successfully processed.

Student: Abdul Subhan
Registration No: FA23-BDS-061
Build: ${BUILD_NUMBER}
Status: SUCCESS

Model training, testing, Docker Hub push, and deployment completed successfully.

Docker Image: ${DOCKER_IMAGE}:${BUILD_NUMBER}
Latest Image: ${DOCKER_IMAGE}:latest

Application: http://${BUILD_URL.split('/')[2].split(':')[0]}:${PORT}/metrics

Jenkins Build: ${BUILD_URL}

Regards,
Jenkins CI/CD Pipeline
""",
                recipientProviders: [developers()]
            )
        }

        failure {
            emailext(
                subject: "FAILED: ${JOB_NAME} #${BUILD_NUMBER}",
                body: """
Hello,

The pipeline triggered by your Git commit has failed.

Student: Abdul Subhan
Registration No: FA23-BDS-061
Build: ${BUILD_NUMBER}
Status: FAILURE

Please check the Jenkins console output.

Jenkins Build: ${BUILD_URL}

Regards,
Jenkins CI/CD Pipeline
""",
                recipientProviders: [developers()]
            )
        }
    }
}
