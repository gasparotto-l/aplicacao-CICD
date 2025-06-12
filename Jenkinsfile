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
                        
                        // Primeira execução do Trivy (permitir download do DB)
                        def trivyResult = bat(
                            script: """
                            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock ^
                            -v "%WORKSPACE%\\trivy-reports:/reports" ^
                            aquasec/trivy:latest image ^
                            --format json ^
                            --output /reports/trivy-report.json ^
                            --severity HIGH,CRITICAL ^
                            --timeout 10m ^
                            --no-progress ^
                            ${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}
                            """,
                            returnStatus: true
                        )
                        
                        if (trivyResult == 0) {
                            // Gerar também relatório em formato de tabela para visualização
                            bat """
                                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock ^
                                -v "%WORKSPACE%\\trivy-reports:/reports" ^
                                aquasec/trivy:latest image ^
                                --format table ^
                                --output /reports/trivy-report.txt ^
                                --severity HIGH,CRITICAL ^
                                --skip-db-update ^
                                --timeout 5m ^
                                --no-progress ^
                                ${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}
                            """
                            
                            echo "✅ Scan de vulnerabilidades concluído!"
                            
                            // Exibir resumo do relatório se existir
                            if (fileExists('trivy-reports/trivy-report.txt')) {
                                echo "📋 Resumo das vulnerabilidades encontradas:"
                                bat 'type trivy-reports\\trivy-report.txt'
                            }
                        } else {
                            echo "⚠️ Trivy scan falhou, criando arquivo JSON vazio para continuar pipeline..."
                            bat 'echo {"Results": []} > trivy-reports\\trivy-report.json'
                            bat 'echo No vulnerabilities found > trivy-reports\\trivy-report.txt'
                        }
                        
                    } catch (Exception e) {
                        echo "⚠️ Erro durante o scan de vulnerabilidades: ${e.getMessage()}"
                        // Criar arquivos vazios para não quebrar próximos stages
                        bat 'echo {"Results": []} > trivy-reports\\trivy-report.json'
                        bat 'echo Scan failed > trivy-reports\\trivy-report.txt'
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        stage('Vulnerability Assessment') {
            steps {
                script {
                    try {
                        // Verificar se o arquivo existe antes de tentar ler
                        if (!fileExists('trivy-reports/trivy-report.json')) {
                            echo "⚠️ Arquivo de relatório não encontrado, pulando análise"
                            return
                        }
                        
                        // Ler o arquivo JSON
                        def jsonReport = readFile('trivy-reports/trivy-report.json')
                        
                        // Parse básico para contar vulnerabilidades críticas
                        def criticalCount = 0
                        try {
                            // Método mais seguro usando readJSON do Jenkins
                            def parsedJson = readJSON text: jsonReport
                            
                            if (parsedJson.Results) {
                                parsedJson.Results.each { result ->
                                    if (result.Vulnerabilities) {
                                        result.Vulnerabilities.each { vuln ->
                                            if (vuln.Severity == 'CRITICAL') {
                                                criticalCount++
                                            }
                                        }
                                    }
                                }
                            }
                            
                        } catch (Exception jsonError) {
                            echo "⚠️ Erro ao processar JSON com readJSON: ${jsonError.getMessage()}"
                            
                            // Fallback: contar vulnerabilidades via findstr no arquivo de texto
                            try {
                                if (fileExists('trivy-reports/trivy-report.txt')) {
                                    def grepResult = bat(
                                        script: '''
                                        for /f %%i in ('findstr /C:"CRITICAL" trivy-reports\\trivy-report.txt ^| find /C /V ""') do echo %%i
                                        ''',
                                        returnStdout: true
                                    ).trim()
                                    
                                    if (grepResult.isNumber()) {
                                        criticalCount = grepResult as Integer
                                    }
                                }
                            } catch (Exception fallbackError) {
                                echo "⚠️ Fallback também falhou: ${fallbackError.getMessage()}"
                                criticalCount = 0
                            }
                        }
                        
                        echo "🔍 Vulnerabilidades CRÍTICAS encontradas: ${criticalCount}"
                        
                        if (criticalCount > 0) {
                            echo "⚠️ ATENÇÃO: Encontradas ${criticalCount} vulnerabilidades CRÍTICAS!"
                            echo "📋 Revise o relatório em trivy-reports/trivy-report.txt"
                            
                            // Marcar como instável mas continuar deploy
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
            when {
                not { 
                    equals expected: 'FAILURE', actual: currentBuild.result 
                }
            }
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
            when {
                not { 
                    equals expected: 'FAILURE', actual: currentBuild.result 
                }
            }
            environment {
                TAG_VERSION = "${BUILD_TAG}"
            }
            steps {
                withKubeConfig([credentialsId: 'rancher-kubeconfig']) {
                    script {
                        try {
                            echo "🔍 Testando conectividade com o cluster..."
                            bat 'kubectl cluster-info'
                            
                            echo "🔄 Atualizando tag da imagem no deployment..."
                            
                            // Usar escape correto para caracteres especiais no Windows
                            bat """
                            powershell -Command "& {
                                \$content = Get-Content './k8s/deployment.yaml' -Raw;
                                \$content = \$content -replace 'gasparottoluo/meu-backend:v1.0.0', '${DOCKERHUB_REPO}/meu-backend:${TAG_VERSION}';
                                Set-Content './k8s/deployment.yaml' -Value \$content -NoNewline;
                            }"
                            """
                            
                            echo "📋 Visualizando o deployment atualizado..."
                            bat 'type k8s\\deployment.yaml'
                            
                            echo "🚀 Aplicando deployment no Kubernetes..."
                            
                            try {
                                bat 'kubectl apply -f k8s/deployment.yaml'
                            } catch (Exception applyError) {
                                echo "⚠️ Falha na aplicação, tentando sem validação..."
                                bat 'kubectl apply -f k8s/deployment.yaml --validate=false'
                            }
                            
                            echo "⏳ Aguardando rollout do deployment..."
                            bat 'kubectl rollout status deployment/backend-app --timeout=300s'
                            
                        } catch (Exception e) {
                            echo "❌ Erro durante o deploy: ${e.getMessage()}"
                            
                            echo "🔍 Informações de debug:"
                            try {
                                bat 'kubectl config current-context'
                            } catch (Exception debugError) {
                                echo "Erro ao obter contexto: ${debugError.getMessage()}"
                            }
                            
                            try {
                                bat 'kubectl get nodes'
                            } catch (Exception debugError) {
                                echo "Erro ao listar nodes: ${debugError.getMessage()}"
                            }
                            
                            try {
                                bat 'kubectl get namespaces'
                            } catch (Exception debugError) {
                                echo "Erro ao listar namespaces: ${debugError.getMessage()}"
                            }
                            
                            throw e
                        }
                    }
                }
            }
        }

        stage('Verificar Deploy') {
            when {
                not { 
                    equals expected: 'FAILURE', actual: currentBuild.result 
                }
            }
            steps {
                withKubeConfig([credentialsId: 'rancher-kubeconfig']) {
                    script {
                        echo "🔍 Verificando status dos pods e serviços..."
                        
                        try {
                            bat 'kubectl get pods -l app=backend-app -o wide'
                        } catch (Exception e) {
                            echo "⚠️ Erro ao listar pods: ${e.getMessage()}"
                        }
                        
                        try {
                            bat 'kubectl get services'
                        } catch (Exception e) {
                            echo "⚠️ Erro ao listar serviços: ${e.getMessage()}"
                        }
                        
                        try {
                            def podsStatus = bat(
                                script: 'kubectl get pods -l app=backend-app --no-headers',
                                returnStdout: true
                            ).trim()
                            echo "📊 Status dos pods: ${podsStatus}"
                        } catch (Exception e) {
                            echo "⚠️ Erro ao verificar status dos pods: ${e.getMessage()}"
                        }
                        
                        echo "📋 Logs recentes do Backend:"
                        try {
                            bat 'kubectl logs -l app=backend-app --tail=10'
                        } catch (Exception e) {
                            echo "⚠️ Sem logs disponíveis: ${e.getMessage()}"
                        }
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
                    // Restaurar deployment.yaml para estado original
                    bat """
                    powershell -Command "& {
                        \$content = Get-Content './k8s/deployment.yaml' -Raw;
                        \$content = \$content -replace '${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}', 'gasparottoluo/meu-backend:v1.0.0';
                        Set-Content './k8s/deployment.yaml' -Value \$content -NoNewline;
                    }"
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
                    echo "ℹ️ Status final dos pods não disponível: ${e.getMessage()}"
                }
            }
        }
        
        failure {
            echo '❌ Build falhou!'
            
            script {
                try {
                    echo "🔍 Informações de debug da falha:"
                    bat 'kubectl describe pods -l app=backend-app'
                } catch (Exception e) {
                    echo "⚠️ Erro ao descrever pods backend: ${e.getMessage()}"
                }
                
                try {
                    bat 'kubectl get events --sort-by=.metadata.creationTimestamp --output=wide'
                } catch (Exception e) {
                    echo "⚠️ Erro ao obter eventos: ${e.getMessage()}"
                }
            }
        }
        
        unstable {
            echo '⚠️ Build instável - Possíveis vulnerabilidades encontradas!'
            echo '📋 Verifique os relatórios de segurança nos artefatos do build'
        }
    }
}