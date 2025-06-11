pipeline {
    agent any

    triggers {
        pollSCM('* * * * *') // Executa a cada minuto (ajuste se necessário)
    }

    environment {
        DOCKERHUB_REPO = "gasparottoluo"       // ✅ CORRIGIDO: usar o username do Docker Hub (não GitHub)
        BUILD_TAG = "${env.BUILD_ID}"          // Jenkins Build ID como versão
    }

    stages {
        stage('Build Frontend Docker Image') {
            steps {
                script {
                    def frontendapp = docker.build("${DOCKERHUB_REPO}/meu-frontend:${BUILD_TAG}", "-f ./frontend/Dockerfile ./frontend")
                }
            }
        }

        stage('Build Backend Docker Image') {
            steps {
                script {
                    def backendapp = docker.build("${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}", "-f ./backend/Dockerfile ./backend")
                }
            }
        }

        stage('Push Docker Images') {
            parallel {
                stage('Push Frontend') {
                    steps {
                        script {
                            def frontendapp = docker.image("${DOCKERHUB_REPO}/meu-frontend:${BUILD_TAG}")
                            docker.withRegistry('https://index.docker.io/v1/', 'docker-creds') {
                                frontendapp.push('latest')
                                frontendapp.push("${BUILD_TAG}")
                            }
                        }
                    }
                }
                stage('Push Backend') {
                    steps {
                        script {
                            def backendapp = docker.image("${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}")
                            docker.withRegistry('https://index.docker.io/v1/', 'docker-creds') {
                                backendapp.push('latest')
                                backendapp.push("${BUILD_TAG}")
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy no Kubernetes') {
            environment {
                tag_version = "${BUILD_TAG}"
            }
            steps {
                withKubeConfig([credentialsId: 'rancher-kubeconfig', serverUrl: 'https://192.168.1.81:6443']) {
                    script {
                        // Substitui as imagens no YAML usando PowerShell
                        bat """
                            powershell -Command "(Get-Content ./k8s/deployment.yaml) -replace 'meu-frontend:v1.0.0', '${DOCKERHUB_REPO}/meu-frontend:${tag_version}' | Set-Content ./k8s/deployment.yaml"
                            powershell -Command "(Get-Content ./k8s/deployment.yaml) -replace 'meu-backend:v1.0.0', '${DOCKERHUB_REPO}/meu-backend:${tag_version}' | Set-Content ./k8s/deployment.yaml"
                        """
                        // Aplica e espera rollout
                        bat 'kubectl apply -f k8s/deployment.yaml'
                        bat 'kubectl rollout status deployment/frontend-app'
                        bat 'kubectl rollout status deployment/backend-app'
                    }
                }
            }
        }

        stage('Verificar Deploy') {
            steps {
                withKubeConfig([credentialsId: 'rancher-kubeconfig', serverUrl: 'https://192.168.1.81:6443']) {
                    bat 'kubectl get pods -l app=frontend-app'
                    bat 'kubectl get pods -l app=backend-app'
                    bat 'kubectl get services'
                }
            }
        }
    }

    post {
        always {
            chuckNorris()
        }
        success {
            echo '🚀 Deploy realizado com sucesso!'
            echo '✅ Frontend: ' + "${DOCKERHUB_REPO}/meu-frontend:${BUILD_TAG}"
            echo '✅ Backend: ' + "${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}"
            echo '🌐 Frontend disponível em: http://localhost:30000'
            echo '🔧 Backend disponível em: http://localhost:30081'
        }
        failure {
            echo '❌ Build falhou!'
        }
        unstable {
            echo '⚠️ Build instável!'
        }
    }
}