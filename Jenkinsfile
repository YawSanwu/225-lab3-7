
pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'roseaw-dockerhub'
        DOCKER_IMAGE = 'cithit/sanwuya'
        IMAGE_TAG = "lab3-7-build-${BUILD_NUMBER}"
        GITHUB_URL = 'https://github.com/YawSanwu/225-lab3-7.git'
        KUBECONFIG = credentials('sanwuya-225')
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: "${GITHUB_URL}"]]
                ])
            }
        }

        stage('Lint HTML') {
            steps {
                sh 'npm install htmlhint --save-dev'
                sh 'npx htmlhint *.html'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.withRegistry(
                        'https://registry-1.docker.io',
                        "${DOCKER_CREDENTIALS_ID}"
                    ) {
                        docker.build(
                            "${DOCKER_IMAGE}:${IMAGE_TAG}",
                            "-f Dockerfile.build ."
                        )
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        "${DOCKER_CREDENTIALS_ID}"
                    ) {
                        docker.image(
                            "${DOCKER_IMAGE}:${IMAGE_TAG}"
                        ).push()
                    }
                }
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                sh """
                    sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|g' deployment-dev.yaml
                    kubectl apply -f deployment-dev.yaml
                """
            }
        }

        stage('Wait for Dev Deployment') {
            steps {
                sh 'kubectl rollout status deployment/sanwuya-lab3-7-dev-deployment --timeout=180s'
            }
        }

        stage('Run Acceptance Tests') {
            steps {
                sh '''
                    docker rm -f qa-tests-lab3-7 || true
                    docker build -t qa-tests-lab3-7 -f Dockerfile.test .
                    docker run --name qa-tests-lab3-7 qa-tests-lab3-7
                '''
            }
            post {
                always {
                    sh 'docker rm -f qa-tests-lab3-7 || true'
                }
            }
        }

        stage('Run Security Checks') {
            steps {
                sh 'docker pull public.ecr.aws/portswigger/dastardly:latest'
                sh '''
                    docker run --user $(id -u) \
                    -v "${WORKSPACE}:${WORKSPACE}:rw" \
                    -e BURP_START_URL=http://10.48.229.154/lab3-7/ \
                    -e BURP_REPORT_FILE_PATH="${WORKSPACE}/dastardly-report.xml" \
                    public.ecr.aws/portswigger/dastardly:latest
                '''
            }
        }

        stage('Deploy to Prod Environment') {
            when {
                expression { false }
            }
            steps {
                sh 'kubectl apply -f deployment-prod.yaml'
            }
        }

        stage('Check Kubernetes Cluster') {
            steps {
                sh 'kubectl get all'
            }
        }
    }

    post {
        always {
            junit testResults: 'dastardly-report.xml',
                  allowEmptyResults: true,
                  skipPublishingChecks: true
        }
        success {
            slackSend color: "good",
                message: "Build Completed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        unstable {
            slackSend color: "warning",
                message: "Build Unstable: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        failure {
            slackSend color: "danger",
                message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
    }
}
