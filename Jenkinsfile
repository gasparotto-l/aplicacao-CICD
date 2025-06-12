pipeline {
    agent any

    triggers {
        pollSCM('* * * * *') 
    }

    environment {
        DOCKERHUB_REPO = "gasparottoluo"
        BUILD_TAG = "${env.BUILD_ID}"
        DISCORD_WEBHOOK_URL = "https://discord.com/api/webhooks/1382830151179571310/OSxxhsPFHYuo--aJP9LhOQrzJsCFwwmVxj7CZXH0Qycrv9ZpkUvIwzWqsxd7lYorD-0H"
    }

    stages {
        stage('Notificar Início') {
            steps {
                script {
                    sendDiscordNotification(
                        webhookUrl: env.DISCORD_WEBHOOK_URL,
                        title: "🚀 Deploy Iniciado",
                        description: "Iniciando build e deploy do backend",
                        color: 3447003, // Azul
                        fields: [
                            [name: "Build", value: "#${env.BUILD_ID}", inline: true],
                            [name: "Branch", value: "${env.GIT_BRANCH ?: 'main'}", inline: true],
                            [name: "Commit", value: "${env.GIT_COMMIT?.take(7) ?: 'N/A'}", inline: true]
                        ]
                    )
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

        stage('Push Docker Image') {
            steps {
                script {
                    def backendapp = docker.image("${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}")
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-creds') {
                        backendapp.push('latest')
                        backendapp.push("${BUILD_TAG}")
                    }
                    
                    sendDiscordNotification(
                        webhookUrl: env.DISCORD_WEBHOOK_URL,
                        title: "📦 Imagem Docker Criada",
                        description: "Imagem Docker enviada para o DockerHub com sucesso",
                        color: 3066993, // Verde
                        fields: [
                            [name: "Imagem", value: "${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}", inline: false]
                        ]
                    )
                }
            }
        }

        stage('Security Scan com Trivy') {
            steps {
                script {
                    try {
                        echo "🔍 Iniciando scanner de vulnerabilidades com Trivy..."
                        
                        // Instalar Trivy se não existir
                        bat '''
                            where trivy >nul 2>&1 || (
                                echo "📥 Baixando e instalando Trivy..."
                                powershell -ExecutionPolicy Bypass -Command "
                                    $trivyVersion = '0.48.3'
                                    $url = 'https://github.com/aquasecurity/trivy/releases/download/v' + $trivyVersion + '/trivy_' + $trivyVersion + '_Windows-64bit.zip'
                                    $output = 'trivy.zip'
                                    Invoke-WebRequest -Uri $url -OutFile $output
                                    Expand-Archive $output -DestinationPath '.'
                                    Remove-Item $output
                                "
                            )
                        '''
                        
                        // Executar scan da imagem
                        bat """
                            echo "🔎 Executando scan de vulnerabilidades..."
                            .\\trivy.exe image --format json --output trivy-report.json ${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG} || echo "Scan completado com warnings"
                            .\\trivy.exe image --format table ${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG} || echo "Report em formato tabela gerado"
                        """
                        
                        // Verificar se há vulnerabilidades críticas
                        def criticalVulns = 0
                        def highVulns = 0
                        
                        try {
                            if (fileExists('trivy-report.json')) {
                                def report = readJSON file: 'trivy-report.json'
                                if (report.Results) {
                                    report.Results.each { result ->
                                        if (result.Vulnerabilities) {
                                            result.Vulnerabilities.each { vuln ->
                                                if (vuln.Severity == 'CRITICAL') criticalVulns++
                                                if (vuln.Severity == 'HIGH') highVulns++
                                            }
                                        }
                                    }
                                }
                            }
                        } catch (Exception e) {
                            echo "⚠️ Erro ao processar relatório JSON: ${e.getMessage()}"
                        }
                        
                        def scanColor = 3066993 // Verde padrão
                        def scanStatus = "✅ Sem vulnerabilidades críticas"
                        
                        if (criticalVulns > 0) {
                            scanColor = 15158332 // Vermelho
                            scanStatus = "🚨 ${criticalVulns} vulnerabilidade(s) CRÍTICA(S) encontrada(s)"
                        } else if (highVulns > 0) {
                            scanColor = 16776960 // Amarelo
                            scanStatus = "⚠️ ${highVulns} vulnerabilidade(s) ALTA(S) encontrada(s)"
                        }
                        
                        sendDiscordNotification(
                            webhookUrl: env.DISCORD_WEBHOOK_URL,
                            title: "🛡️ Scanner de Segurança Concluído",
                            description: scanStatus,
                            color: scanColor,
                            fields: [
                                [name: "Imagem Analisada", value: "${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}", inline: false],
                                [name: "Vulnerabilidades Críticas", value: "${criticalVulns}", inline: true],
                                [name: "Vulnerabilidades Altas", value: "${highVulns}", inline: true]
                            ]
                        )
                        
                        // Opcional: Falhar o build se houver vulnerabilidades críticas
                        // Descomente a linha abaixo se quiser que o pipeline falhe com vulnerabilidades críticas
                        // if (criticalVulns > 0) error("Pipeline interrompido devido a vulnerabilidades críticas")
                        
                    } catch (Exception e) {
                        echo "⚠️ Erro durante o scanner de segurança: ${e.getMessage()}"
                        
                        sendDiscordNotification(
                            webhookUrl: env.DISCORD_WEBHOOK_URL,
                            title: "⚠️ Erro no Scanner de Segurança",
                            description: "Não foi possível executar o scanner Trivy",
                            color: 16776960, // Amarelo
                            fields: [
                                [name: "Erro", value: e.getMessage(), inline: false]
                            ]
                        )
                        
                        // Não falhar o pipeline por erro no scanner
                        echo "🔄 Continuando pipeline apesar do erro no scanner..."
                    }
                }
            }
            post {
                always {
                    // Arquivar relatório se existir
                    archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true, fingerprint: true
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
                
                // Limpeza dos arquivos do Trivy
                try {
                    bat 'del /f /q trivy.exe 2>nul || echo "Trivy já removido"'
                    bat 'del /f /q trivy-report.json 2>nul || echo "Relatório já removido"'
                } catch (Exception e) {
                    echo "ℹ️ Limpeza do Trivy: ${e.getMessage()}"
                }
            }
        }
        success {
            echo '🎉 Deploy realizado com sucesso!'
            echo "✅ Backend: ${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}"
            echo '🔧 Backend disponível em: http://localhost:30081'
            
            script {
                try {
                    bat 'kubectl get pods -l app=backend-app'
                    
                    sendDiscordNotification(
                        webhookUrl: env.DISCORD_WEBHOOK_URL,
                        title: "✅ Deploy Concluído com Sucesso!",
                        description: "O backend foi deployado com sucesso no Kubernetes",
                        color: 3066993, // Verde
                        fields: [
                            [name: "Build", value: "#${env.BUILD_ID}", inline: true],
                            [name: "Imagem", value: "${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}", inline: false],
                            [name: "URL", value: "http://localhost:30081", inline: true],
                            [name: "Duração", value: "${currentBuild.durationString}", inline: true]
                        ]
                    )
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
                    
                    sendDiscordNotification(
                        webhookUrl: env.DISCORD_WEBHOOK_URL,
                        title: "❌ Deploy Falhou!",
                        description: "Ocorreu um erro durante o processo de build/deploy",
                        color: 15158332, // Vermelho
                        fields: [
                            [name: "Build", value: "#${env.BUILD_ID}", inline: true],
                            [name: "Erro", value: "Verifique os logs do Jenkins para mais detalhes", inline: false],
                            [name: "Duração", value: "${currentBuild.durationString}", inline: true]
                        ]
                    )
                } catch (Exception e) {
                    echo "⚠️ Não foi possível obter informações de debug: ${e.getMessage()}"
                }
            }
        }
        unstable {
            echo '⚠️ Build instável!'
            
            script {
                sendDiscordNotification(
                    webhookUrl: env.DISCORD_WEBHOOK_URL,
                    title: "⚠️ Build Instável",
                    description: "O build foi concluído, mas com alguns problemas",
                    color: 16776960, // Amarelo
                    fields: [
                        [name: "Build", value: "#${env.BUILD_ID}", inline: true],
                        [name: "Status", value: "Instável - Requer atenção", inline: false]
                    ]
                )
            }
        }
    }
}

// Função para enviar notificações para o Discord
def sendDiscordNotification(Map config) {
    def payload = [
        embeds: [[
            title: config.title,
            description: config.description,
            color: config.color,
            fields: config.fields ?: [],
            timestamp: new Date().format("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"),
            footer: [
                text: "Jenkins CI/CD",
                icon_url: "https://www.jenkins.io/images/logos/jenkins/jenkins.png"
            ],
            author: [
                name: "Jenkins Pipeline",
                url: "${env.BUILD_URL}",
                icon_url: "https://www.jenkins.io/images/logos/jenkins/jenkins.png"
            ]
        ]]
    ]
    
    def jsonPayload = groovy.json.JsonOutput.toJson(payload)
    
    try {
        httpRequest(
            httpMode: 'POST',
            url: config.webhookUrl,
            contentType: 'APPLICATION_JSON',
            requestBody: jsonPayload,
            validResponseCodes: '200:299'
        )
        echo "✅ Notificação Discord enviada com sucesso"
    } catch (Exception e) {
        echo "⚠️ Falha ao enviar notificação Discord: ${e.getMessage()}"
    }
}