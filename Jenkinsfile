
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
        STUDENT_ID = "FA23-BDS-061"
    }

    stages {
        stage('Checkout and Identify Commit Author') {
            steps {
                checkout scm

                script {
                    env.COMMIT_AUTHOR_EMAIL = sh(
                        returnStdout: true,
                        script: 'git log -1 --format=%ae'
                    ).trim()

                    if (!(env.COMMIT_AUTHOR_EMAIL ==~ /[^@\s]+@[^@\s]+\.[^@\s]+/)) {
                        error("Invalid or missing commit author email")
                    }

                    echo "Commit author: ${env.COMMIT_AUTHOR_EMAIL}"
                    echo "Commit SHA: ${env.GIT_COMMIT}"
                }
            }
        }

        stage('Validate and Build Model') {
            steps {
                sh '''
                    set -eu

                    test -f config.json
                    test -f configs/FA23-BDS-061.json
                    test -f dataset/train.csv
                    test -f train.py
                    test -f app.py

                    docker build \
                        --build-arg GIT_COMMIT="${GIT_COMMIT}" \
                        -t ${IMAGE}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Test Candidate') {
            steps {
                sh '''
                    set -eu

                    docker rm -f student-ml-candidate >/dev/null 2>&1 || true

                    docker run -d \
                        --name student-ml-candidate \
                        -p 8001:8000 \
                        ${IMAGE}:${BUILD_NUMBER}

                    READY=0

                    for i in $(seq 1 30); do
                        if curl -fsS http://localhost:8001/health >/dev/null; then
                            READY=1
                            break
                        fi
                        sleep 2
                    done

                    test "$READY" -eq 1

                    curl -fsS http://localhost:8001/metrics > candidate-metrics.json

                    python3 -c "
import json, os
m = json.load(open('candidate-metrics.json'))
assert m['student_id'] == 'FA23-BDS-061', m
assert m['training_rows'] >= 100, m
assert m['accuracy'] >= 0 and m['accuracy'] <= 1, m
assert m['git_sha'] == os.environ['GIT_COMMIT'], m
assert m['model_version'] != 'local', m
print('Candidate model verified:', json.dumps(m, indent=2))
"

                    curl -fsS -X POST \
                        http://localhost:8001/predict \
                        -H 'Content-Type: application/json' \
                        -d '{"records":[{"pressure_loss":0.5,"flow_change":0.2,"valve_age":0.3,"sensor_noise":0.1}]}'

                    echo
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
                        set -eu

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

        stage('Deploy with Rollback') {
            steps {
                sh '''
                    set -eu

                    OLD="${CONTAINER}-previous"

                    docker rm -f "$OLD" >/dev/null 2>&1 || true

                    if docker inspect ${CONTAINER} >/dev/null 2>&1; then
                        docker stop ${CONTAINER}
                        docker rename ${CONTAINER} "$OLD"
                    fi

                    if ! docker run -d \
                        --restart unless-stopped \
                        --name ${CONTAINER} \
                        -p ${PORT}:8000 \
                        ${DOCKER_IMAGE}:${BUILD_NUMBER}; then

                        docker rm -f ${CONTAINER} >/dev/null 2>&1 || true

                        if docker inspect "$OLD" >/dev/null 2>&1; then
                            docker rename "$OLD" ${CONTAINER}
                            docker start ${CONTAINER}
                        fi

                        exit 1
                    fi

                    READY=0

                    for i in $(seq 1 30); do
                        if curl -fsS http://localhost:${PORT}/health >/dev/null; then
                            READY=1
                            break
                        fi
                        sleep 2
                    done

                    if [ "$READY" -ne 1 ]; then
                        docker rm -f ${CONTAINER} >/dev/null 2>&1 || true

                        if docker inspect "$OLD" >/dev/null 2>&1; then
                            docker rename "$OLD" ${CONTAINER}
                            docker start ${CONTAINER}
                        fi

                        exit 1
                    fi

                    curl -fsS http://localhost:${PORT}/metrics > deployed-metrics.json

                    python3 -c "
import json, os
m = json.load(open('deployed-metrics.json'))
assert m['student_id'] == 'FA23-BDS-061', m
assert m['git_sha'] == os.environ['GIT_COMMIT'], m
assert m['training_rows'] >= 100, m
print('Deployment verified:', json.dumps(m, indent=2))
"

                    docker rm -f "$OLD" >/dev/null 2>&1 || true
                '''
            }
        }
    }

    post {
        success {
            script {
                if (env.COMMIT_AUTHOR_EMAIL) {
                    def emailRecipients = env.COMMIT_AUTHOR_EMAIL.equalsIgnoreCase('qasimalik@gmail.com') ?
                        env.COMMIT_AUTHOR_EMAIL :
                        "${env.COMMIT_AUTHOR_EMAIL}, qasimalik@gmail.com"

                    emailext(
                        to: emailRecipients,
                        subject: "SUCCESS: FA23-BDS-061 - ${JOB_NAME} #${BUILD_NUMBER}",
                        body: """
Hello,

Your GitHub commit was processed successfully.

Student: Abdul Subhan
Registration: FA23-BDS-061
Commit: ${GIT_COMMIT}
Build: ${BUILD_NUMBER}
Status: SUCCESS

Model retraining, candidate tests, Docker Hub push and EC2 deployment completed.

Docker image: ${DOCKER_IMAGE}:${BUILD_NUMBER}
Metrics: http://13.62.2.110:8000/metrics
Jenkins build: ${BUILD_URL}

Regards,
Jenkins CI/CD Pipeline
"""
                    )
                } else {
                    echo "No valid commit author email; notification skipped."
                }
            }
        }

        failure {
            script {
                if (env.COMMIT_AUTHOR_EMAIL) {
                    def emailRecipients = env.COMMIT_AUTHOR_EMAIL.equalsIgnoreCase('qasimalik@gmail.com') ?
                        env.COMMIT_AUTHOR_EMAIL :
                        "${env.COMMIT_AUTHOR_EMAIL}, qasimalik@gmail.com"

                    emailext(
                        to: emailRecipients,
                        subject: "FAILED: FA23-BDS-061 - ${JOB_NAME} #${BUILD_NUMBER}",
                        body: """
Hello,

Your GitHub commit triggered a pipeline that failed.

Student: Abdul Subhan
Registration: FA23-BDS-061
Commit: ${GIT_COMMIT}
Build: ${BUILD_NUMBER}
Status: FAILURE

Please inspect the Jenkins console output.

Jenkins build: ${BUILD_URL}

Regards,
Jenkins CI/CD Pipeline
"""
                    )
                } else {
                    echo "No valid commit author email; notification skipped."
                }
            }
        }

        always {
            sh 'docker rm -f student-ml-candidate >/dev/null 2>&1 || true'
        }
    }
}
