pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'production'], description: 'Target Environment')
        string(name: 'IMAGE_TAG', defaultValue: 'v1.0', description: 'Docker Image Tag')
    }

    environment {
        DOCKER_USER = 'tanshul12'
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
                        bat 'findstr /C:"Smart Campus Energy Monitor" index.html'
                    }
                }
                stage('Validate Deployment Manifest') {
                    steps {
                        bat 'findstr /C:"campus-energy-monitor" deployment.yaml'
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
                bat """
                    docker build -t %IMAGE_NAME%:%IMAGE_TAG% .
                    docker run -d --name test-container -p 8089:80 %IMAGE_NAME%:%IMAGE_TAG%
                    timeout /t 3 /nobreak
                    curl -f http://localhost:8089 || (docker rm -f test-container && exit 1)
                    docker rm -f test-container
                """
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat """
                        echo %DH_PASS% | docker login -u %DH_USER% --password-stdin
                        docker push %IMAGE_NAME%:%IMAGE_TAG%
                        docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                        docker push %IMAGE_NAME%:latest
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
                    bat """
                        kubectl --kubeconfig="%KUBECONFIG%" apply -f deployment.yaml
                        kubectl --kubeconfig="%KUBECONFIG%" rollout status deployment/campus-energy-monitor --timeout=60s
                    """
                }
            }
        }
    }
}
