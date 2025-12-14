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

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Source Code') {
            steps {
                git branch: 'main', url: "${GIT_APP_REPO}"
            }
        }

        stage('Build Backend & Deploy Nexus') {
            steps {
                dir('movieApi') {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-cred',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh '''
#!/bin/bash
set -e

echo ">>> Génération settings.xml"
cat > settings.xml <<EOF
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
                    sh '''
#!/bin/bash
set -e

echo ">>> Installation dépendances frontend"
npm install

echo ">>> Build frontend"
npm run build
'''
                }
            }
        }

        stage('Build & Push Backend Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
#!/bin/bash
set -e

echo ">>> Build image backend"
docker build -t ${BACKEND_IMAGE} ./movieApi

DOCKER_HOST=$(echo ${BACKEND_IMAGE} | cut -d/ -f1)

echo ">>> Login Docker registry"
echo "$DOCKER_PASS" | docker login "$DOCKER_HOST" -u "$DOCKER_USER" --password-stdin

docker push ${BACKEND_IMAGE}
docker logout "$DOCKER_HOST"
'''
                }
            }
        }

        stage('Build & Push Frontend Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
#!/bin/bash
set -e

echo ">>> Build image frontend"
docker build -t ${FRONTEND_IMAGE} ./movieUi

DOCKER_HOST=$(echo ${FRONTEND_IMAGE} | cut -d/ -f1)

echo ">>> Login Docker registry"
echo "$DOCKER_PASS" | docker login "$DOCKER_HOST" -u "$DOCKER_USER" --password-stdin

docker push ${FRONTEND_IMAGE}
docker logout "$DOCKER_HOST"
'''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                dir("${DOCKER_COMPOSE_DIR}") {
                    sh '''
#!/bin/bash
set -e

echo ">>> Déploiement Docker Compose"
docker-compose pull
docker-compose up -d
'''
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline exécuté avec succès"
        }
        failure {
            echo "❌ Pipeline en échec – vérifier les logs"
        }
        always {
            cleanWs()
        }
    }
}
