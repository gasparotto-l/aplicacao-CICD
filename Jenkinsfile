pipeline {
    agent any

    triggers {
        pollSCM('* * * * *') 
    }

    environment {
        DOCKERHUB_REPO = "gasparottoluo"
        BUILD_TAG = "${env.BUILD_ID}"
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
                withKubeConfig([credentialsId: 'rancher-kubeconfig']) {
                    script {
                        try {
                            echo "🔍 Testando conectividade com o cluster..."
                            bat 'kubectl cluster-info'
                            
                            echo "🔄 Atualizando tags das imagens no deployment..."
                            
                            bat """
                                powershell -Command "(Get-Content ./k8s/deployment.yaml) -replace 'gasparottoluo/meu-frontend:v1.0.0', '${DOCKERHUB_REPO}/meu-frontend:${tag_version}' | Set-Content ./k8s/deployment.yaml"
                                powershell -Command "(Get-Content ./k8s/deployment.yaml) -replace 'gasparottoluo/meu-backend:v1.0.0', '${DOCKERHUB_REPO}/meu-backend:${tag_version}' | Set-Content ./k8s/deployment.yaml"
                            """
                            
                            echo "📋 Visualizando o deployment atualizado..."
                            bat 'type k8s\\deployment.yaml'
                            
                            echo "🚀 Aplicando deployment no Kubernetes..."
                            
                            try {
                                bat 'kubectl apply -f k8s/deployment.yaml'
                            } catch (Exception e) {
                                echo "⚠️ Falha na validação, tentando sem validação..."
                                bat 'kubectl apply -f k8s/deployment.yaml --validate=false'
                            }
                            
                            echo "⏳ Aguardando rollout dos deployments..."
                            bat 'kubectl rollout status deployment/frontend-app --timeout=300s'
                            bat 'kubectl rollout status deployment/backend-app --timeout=300s'
                            
                        } catch (Exception e) {
                            echo "❌ Erro durante o deploy: ${e.getMessage()}"
                            
                            
                            echo "🔍 Informações de debug:"
                            bat 'kubectl config current-context || echo "Erro ao obter contexto"'
                            bat 'kubectl get nodes || echo "Erro ao listar nodes"'
                            bat 'kubectl get namespaces || echo "Erro ao listar namespaces"'
                            
                            throw e
                        }
                    }
                }
            }
        }

        stage('Verificar Deploy') {
            steps {
                withKubeConfig([credentialsId: 'rancher-kubeconfig']) {
                    script {
                        echo "🔍 Verificando status dos pods e serviços..."
                        
                        bat 'kubectl get pods -l app=frontend-app -o wide'
                        bat 'kubectl get pods -l app=backend-app -o wide'
                        bat 'kubectl get services'
                        
                        
                        try {
                            bat 'kubectl get pods -l app=frontend-app --no-headers'
                            bat 'kubectl get pods -l app=backend-app --no-headers'
                        } catch (Exception e) {
                            echo "⚠️ Erro ao verificar status dos pods: ${e.getMessage()}"
                        }
                        
                        
                        echo "📋 Logs recentes do Frontend:"
                        bat 'kubectl logs -l app=frontend-app --tail=10 || echo "Sem logs disponíveis"'
                        
                        echo "📋 Logs recentes do Backend:"  
                        bat 'kubectl logs -l app=backend-app --tail=10 || echo "Sem logs disponíveis"'
                    }
                }
            }
        }
    }

    post {
        always {
            chuckNorris()
            
            // Cleanup: Restaura o deployment.yaml original
            script {
                try {
                    bat """
                        powershell -Command "(Get-Content ./k8s/deployment.yaml) -replace '${DOCKERHUB_REPO}/meu-frontend:${BUILD_TAG}', 'gasparottoluo/meu-frontend:v1.0.0' | Set-Content ./k8s/deployment.yaml"
                        powershell -Command "(Get-Content ./k8s/deployment.yaml) -replace '${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}', 'gasparottoluo/meu-backend:v1.0.0' | Set-Content ./k8s/deployment.yaml"
                    """
                    echo "🧹 Deployment.yaml restaurado para o estado original"
                } catch (Exception e) {
                    echo "⚠️ Não foi possível restaurar o deployment.yaml: ${e.getMessage()}"
                }
            }
        }
        success {
            echo '🎉 Deploy realizado com sucesso!'
            echo "✅ Frontend: ${DOCKERHUB_REPO}/meu-frontend:${BUILD_TAG}"
            echo "✅ Backend: ${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}"
            echo '🌐 Frontend disponível em: http://localhost:30000'
            echo '🔧 Backend disponível em: http://localhost:30081'
            
            
            script {
                try {
                    bat 'kubectl get pods -l app=frontend-app'
                    bat 'kubectl get pods -l app=backend-app'
                } catch (Exception e) {
                    echo "ℹ️ Status final dos pods não disponível"
                }
            }
        }
        failure {
            echo '❌ Build falhou!'
            
            
            script {
                try {
                    echo "🔍 Informações de debug da falha:"
                    bat 'kubectl describe pods -l app=frontend-app || echo "Erro ao descrever pods frontend"'
                    bat 'kubectl describe pods -l app=backend-app || echo "Erro ao descrever pods backend"'
                    bat 'powershell -Command "kubectl get events --sort-by=.metadata.creationTimestamp | Select-Object -Last 10" || echo "Erro ao obter eventos"'
                } catch (Exception e) {
                    echo "⚠️ Não foi possível obter informações de debug: ${e.getMessage()}"
                }
            }
        }
        unstable {
            echo '⚠️ Build instável!'
        }
    }
}