pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'production'], description: 'Target Environment')
        string(name: 'IMAGE_TAG', defaultValue: 'v1.0', description: 'Docker Image Tag')
    }

    environment {
        DOCKER_USER = 'Tanshul02'
        IMAGE_NAME  = "${DOCKER_USER}/campus-energy-monitor"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Static Tests & Verification') {
            parallel {
                stage('Validate HTML') {
                    steps {
                        sh 'grep -q "Smart Campus Energy Monitor" index.html'
                    }
                }
                stage('Validate Deployment Manifest') {
                    steps {
                        sh 'grep -q "campus-energy-monitor" deployment.yaml'
                    }
                }
            }
        }

        stage('Archive Configs') {
            steps {
                archiveArtifacts artifacts: 'deployment.yaml, index.html', fingerprint: true
            }
        }

        stage('Build & Test Local Container') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${params.IMAGE_TAG} .
                    docker run -d --name test-container -p 8085:80 ${IMAGE_NAME}:${params.IMAGE_TAG}
                    sleep 3
                    curl -f http://localhost:8085 || (docker rm -f test-container && exit 1)
                    docker rm -f test-container
                """
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo "\$DH_PASS" | docker login -u "\$DH_USER" --password-stdin
                        docker push ${IMAGE_NAME}:${params.IMAGE_TAG}
                        docker tag ${IMAGE_NAME}:${params.IMAGE_TAG} ${IMAGE_NAME}:latest
                        docker push ${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { return params.DEPLOY_ENV == 'production' }
            }
            steps {
                withCredentials([file(credentialsId: 'k8s-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                        kubectl --kubeconfig=\$KUBECONFIG apply -f deployment.yaml
                        if ! kubectl --kubeconfig=\$KUBECONFIG rollout status deployment/campus-energy-monitor --timeout=60s; then
                            echo "Rollout failed, rolling back..."
                            kubectl --kubeconfig=\$KUBECONFIG rollout undo deployment/campus-energy-monitor
                            exit 1
                        fi
                    """
                }
            }
        }
    }
}
