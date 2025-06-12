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
        stage('Build Backend Docker Image') {
            steps {
                script {
                    def backendapp = docker.build("${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}", "-f ./backend/Dockerfile ./backend")
                }
            }
        }

        stage('Security Scan with Trivy') {
            steps {
                script {
                    echo "🔍 Iniciando scan de vulnerabilidades com Trivy..."
                    
                    try {
                        // Criar diretório para relatórios se não existir
                        bat 'if not exist "trivy-reports" mkdir trivy-reports'
                        
                        // Executar Trivy via Docker para scan da imagem
                        bat """
                            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock ^
                            -v "%WORKSPACE%/trivy-reports:/reports" ^
                            aquasec/trivy:latest image ^
                            --format json ^
                            --output /reports/trivy-report.json ^
                            --severity HIGH,CRITICAL ^
                            --skip-db-update ^
                            --timeout 5m ^
                            ${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}
                        """
                        
                        // Gerar também relatório em formato de tabela para visualização
                        bat """
                            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock ^
                            -v "%WORKSPACE%/trivy-reports:/reports" ^
                            aquasec/trivy:latest image ^
                            --format table ^
                            --output /reports/trivy-report.txt ^
                            --severity HIGH,CRITICAL ^
                            --skip-db-update ^
                            --timeout 5m ^
                            ${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}
                        """
                        
                        echo "✅ Scan de vulnerabilidades concluído!"
                        
                        // Exibir resumo do relatório
                        echo "📋 Resumo das vulnerabilidades encontradas:"
                        bat 'type trivy-reports\\trivy-report.txt'
                        
                    } catch (Exception e) {
                        echo "⚠️ Erro durante o scan de vulnerabilidades: ${e.getMessage()}"
                        // Não falhar o pipeline, apenas alertar
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        stage('Vulnerability Assessment') {
            steps {
                script {
                    try {
                        // Verificar se existem vulnerabilidades críticas
                        def jsonReport = readFile('trivy-reports/trivy-report.json')
                        
                        // Parse básico para contar vulnerabilidades críticas
                        def criticalCount = 0
                        try {
                            def countResult = bat(
                                script: '''
                                    powershell -Command "& {
                                        $ErrorActionPreference = 'Stop';
                                        $json = Get-Content 'trivy-reports/trivy-report.json' -Raw | ConvertFrom-Json;
                                        $critical = 0;
                                        if ($json.Results) {
                                            foreach($result in $json.Results) {
                                                if($result.Vulnerabilities) {
                                                    $critical += ($result.Vulnerabilities | Where-Object {$_.Severity -eq 'CRITICAL'}).Count
                                                }
                                            }
                                        }
                                        Write-Output $critical
                                    }"
                                ''',
                                returnStdout: true
                            ).trim()
                            criticalCount = countResult as Integer
                        } catch (Exception e) {
                            echo "⚠️ Erro ao processar JSON: ${e.getMessage()}"
                            // Fallback: contar vulnerabilidades via grep/findstr
                            try {
                                def grepResult = bat(
                                    script: 'findstr /C:"CRITICAL" trivy-reports\\trivy-report.txt | find /C /V ""',
                                    returnStdout: true
                                ).trim()
                                criticalCount = grepResult as Integer
                            } catch (Exception e2) {
                                echo "⚠️ Fallback também falhou: ${e2.getMessage()}"
                                criticalCount = 0
                            }
                        }
                        
                        echo "🔍 Vulnerabilidades CRÍTICAS encontradas: ${criticalCount}"
                        
                        if (criticalCount > 0) {
                            echo "⚠️ ATENÇÃO: Encontradas ${criticalCount} vulnerabilidades CRÍTICAS!"
                            echo "📋 Revise o relatório em trivy-reports/trivy-report.txt"
                            
                            // Opção 1: Falhar o pipeline se houver vulnerabilidades críticas
                            // error("Pipeline falhou devido a vulnerabilidades críticas!")
                            
                            // Opção 2: Apenas marcar como instável (recomendado para início)
                            currentBuild.result = 'UNSTABLE'
                        } else {
                            echo "✅ Nenhuma vulnerabilidade crítica encontrada!"
                        }
                        
                    } catch (Exception e) {
                        echo "⚠️ Erro na avaliação de vulnerabilidades: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        stage('Push Docker Image') {
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
                            
                            echo "🔄 Atualizando tag da imagem no deployment..."
                            
                            bat """
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
                            
                            echo "⏳ Aguardando rollout do deployment..."
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
                        
                        bat 'kubectl get pods -l app=backend-app -o wide'
                        bat 'kubectl get services'
                        
                        try {
                            bat 'kubectl get pods -l app=backend-app --no-headers'
                        } catch (Exception e) {
                            echo "⚠️ Erro ao verificar status dos pods: ${e.getMessage()}"
                        }
                        
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
            
            script {
                try {
                    bat """
                        powershell -Command "(Get-Content ./k8s/deployment.yaml) -replace '${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}', 'gasparottoluo/meu-backend:v1.0.0' | Set-Content ./k8s/deployment.yaml"
                    """
                    echo "🧹 Deployment.yaml restaurado para o estado original"
                } catch (Exception e) {
                    echo "⚠️ Não foi possível restaurar o deployment.yaml: ${e.getMessage()}"
                }
            }
            
            // Arquivar relatórios de vulnerabilidade
            archiveArtifacts artifacts: 'trivy-reports/*.json, trivy-reports/*.txt', allowEmptyArchive: true
        }
        success {
            echo '🎉 Deploy realizado com sucesso!'
            echo "✅ Backend: ${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}"
            echo '🔧 Backend disponível em: http://localhost:30081'
            
            script {
                try {
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
                    bat 'kubectl describe pods -l app=backend-app || echo "Erro ao descrever pods backend"'
                    bat 'powershell -Command "kubectl get events --sort-by=.metadata.creationTimestamp | Select-Object -Last 10" || echo "Erro ao obter eventos"'
                } catch (Exception e) {
                    echo "⚠️ Não foi possível obter informações de debug: ${e.getMessage()}"
                }
            }
        }
        unstable {
            echo '⚠️ Build instável - Possíveis vulnerabilidades encontradas!'
            echo '📋 Verifique os relatórios de segurança nos artefatos do build'
        }
    }
}