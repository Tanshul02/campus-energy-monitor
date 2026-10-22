pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'production'], description: 'Target Environment')
        string(name: 'IMAGE_TAG', defaultValue: 'v1.0', description: 'Docker Image Tag')
    }

    environment {
        DOCKER_USER = 'tanshul12'
        IMAGE_NAME  = "${DOCKER_USER}/campus-energy-monitor"
        DOCKER_BIN  = 'C:\\Users\\tanshul\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_BIN = 'C:\\Users\\tanshul\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
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
                    "%DOCKER_BIN%" build -t %IMAGE_NAME%:%IMAGE_TAG% .
                    "%DOCKER_BIN%" run -d --name test-container -p 8089:80 %IMAGE_NAME%:%IMAGE_TAG%
                    ping -n 4 127.0.0.1 >nul
                    curl -f http://localhost:8089 || ("%DOCKER_BIN%" rm -f test-container && exit 1)
                    "%DOCKER_BIN%" rm -f test-container
                """
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat """
                        echo %DH_PASS% | "%DOCKER_BIN%" login -u %DH_USER% --password-stdin
                        "%DOCKER_BIN%" push %IMAGE_NAME%:%IMAGE_TAG%
                        "%DOCKER_BIN%" tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                        "%DOCKER_BIN%" push %IMAGE_NAME%:latest
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
                        "%KUBECTL_BIN%" --kubeconfig="%KUBECONFIG%" apply -f deployment.yaml
                        "%KUBECTL_BIN%" --kubeconfig="%KUBECONFIG%" rollout status deployment/campus-energy-monitor --timeout=60s
                    """
                }
            }
        }
    }
}
