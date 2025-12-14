pipeline {
    agent any

    tools {
        maven 'maven'
        nodejs 'nodejs'
    }

    environment {
        NEXUS_HOST = "192.168.122.48:8081"
        DOCKER_REPO = "192.168.122.48:5001"

        BACKEND_IMAGE  = "${DOCKER_REPO}/backend-app:latest"
        FRONTEND_IMAGE = "${DOCKER_REPO}/frontend-app:latest"

        GIT_APP_REPO = "https://github.com/camou92/fullstack-deploy.git"
        DOCKER_COMPOSE_DIR = "."
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {

        stage('Clean workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout App Repo') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: "${GIT_APP_REPO}"]]
                ])
            }
        }

        stage('Build Backend & Deploy Artifact') {
            steps {
                dir('movieApi') {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-cred',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh '''#!/bin/bash
                        set -e

                        echo ">>> Création du settings.xml Nexus"
                        cat > settings.xml <<'EOF'
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
  </servers>
</settings>
EOF

                        echo ">>> Build et déploiement Maven"
                        mvn clean deploy -s settings.xml -DskipTests
                        '''
                    }
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('movieUi') {
                    sh '''#!/bin/bash
                    set -e
                    npm install
                    npm run build
                    '''
                }
            }
        }

        stage('Build & Push Backend Docker') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''#!/bin/bash
                    set -e

                    docker build -t ${BACKEND_IMAGE} ./movieApi

                    DOCKER_HOST=$(echo ${BACKEND_IMAGE} | cut -d/ -f1)
                    echo "$DOCKER_PASS" | docker login $DOCKER_HOST -u "$DOCKER_USER" --password-stdin

                    docker push ${BACKEND_IMAGE}
                    docker logout $DOCKER_HOST
                    '''
                }
            }
        }

        stage('Build & Push Frontend Docker') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''#!/bin/bash
                    set -e

                    docker build -t ${FRONTEND_IMAGE} ./movieUi

                    DOCKER_HOST=$(echo ${FRONTEND_IMAGE} | cut -d/ -f1)
                    echo "$DOCKER_PASS" | docker login $DOCKER_HOST -u "$DOCKER_USER" --password-stdin

                    docker push ${FRONTEND_IMAGE}
                    docker logout $DOCKER_HOST
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                dir("${DOCKER_COMPOSE_DIR}") {
                    sh '''#!/bin/bash
                    set -e
                    docker-compose pull
                    docker-compose up -d
                    '''
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#tous-camoutech',
                color: '#36a64f',
                message: "🎉 SUCCESS — Backend, Frontend et images Docker déployés 🚀"
            )
        }
        failure {
            slackSend(
                channel: '#tous-camoutech',
                color: '#ff0000',
                message: "❌ ECHEC — Vérifier les logs Jenkins ⚠️"
            )
        }
        always {
            cleanWs()
        }
    }
}
