pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE = "student-ml-midterm"
        CONTAINER = "student-ml-api"
        PORT = "8000"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Train and Build Docker Image') {
            steps {
                sh '''
                    test -f config.json
                    test -f dataset/train.csv
                    docker build --build-arg GIT_COMMIT=${GIT_COMMIT} -t ${IMAGE}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Test Candidate') {
            steps {
                sh '''
                    docker rm -f student-ml-candidate || true
                    docker run -d --name student-ml-candidate -p 8001:8000 ${IMAGE}:${BUILD_NUMBER}
                    for i in $(seq 1 30); do
                      curl -fsS http://localhost:8001/metrics > candidate-metrics.json && break
                      sleep 2
                    done
                    python3 -c "import json; m=json.load(open('candidate-metrics.json')); assert m['student_id']=='FA23-BDS-061'; assert m['training_rows']>=100; print(m)"
                    docker rm -f student-ml-candidate
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker rm -f ${CONTAINER} || true
                    docker run -d --restart unless-stopped --name ${CONTAINER} -p ${PORT}:8000 ${IMAGE}:${BUILD_NUMBER}
                    sleep 5
                    curl -fsS http://localhost:${PORT}/metrics
                '''
            }
        }
    }

    post {
        success {
            emailext(
                to: 'qasimalik@gmail.com',
                subject: 'SUCCESS - FA23-BDS-061 - ML pipeline',
                body: "Student Reg No: FA23-BDS-061\nBuild: ${BUILD_NUMBER}\nModel deployed successfully.\n"
            )
        }
        failure {
            emailext(
                to: 'qasimalik@gmail.com',
                subject: 'FAILED - FA23-BDS-061 - ML pipeline',
                body: "Student Reg No: FA23-BDS-061\nBuild: ${BUILD_NUMBER}\nCheck Jenkins console output."
            )
        }
    }
}
