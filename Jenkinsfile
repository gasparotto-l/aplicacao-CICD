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
                    frontendapp = docker.build("${DOCKERHUB_REPO}/meu-frontend:${BUILD_TAG}", "-f ./frontend/Dockerfile ./frontend")
                }
            }
        }

        stage('Build Backend Docker Image') {
            steps {
                script {
                    backendapp = docker.build("${DOCKERHUB_REPO}/meu-backend:${BUILD_TAG}", "-f ./backend/Dockerfile ./backend")
                }
            }
        }

        stage('Push Docker Images') {
            parallel {
                stage('Push Frontend') {
                    steps {
                        script {
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
                        // Substitui as imagens no YAML
                        sh """
                            sed -i 's|meu-frontend:v1.0.0|${DOCKERHUB_REPO}/meu-frontend:${tag_version}|g' ./k8s/deployment.yaml
                            sed -i 's|meu-backend:v1.0.0|${DOCKERHUB_REPO}/meu-backend:${tag_