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

        /* =========================
           CLEAN
        ========================== */
        stage('Clean workspace') {
            steps {
                cleanWs()
            }
        }

        /* =========================
           CHECKOUT
        ========================== */
        stage('Checkout App Repo') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: "${GIT_APP_REPO}"]]
                ])
            }
        }

        /* =========================
           BACKEND – MAVEN BUILD
        ========================== */
        stage('Build Backend & Deploy Artifact') {
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

echo ">>> Création settings.xml Nexus"
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

echo ">>> Build JAR Spring Boot"
mvn clean package -DskipTests

echo ">>> Vérification JAR"
ls -lh target/*.jar

echo ">>> Déploiement artifact vers Nexus"
mvn deploy -s settings.xml -DskipTests
'''
                    }
                }
            }
        }

        /* =========================
           FRONTEND – BUILD
        ========================== */
        stage('Build Frontend') {
            steps {
                dir('movieUi') {
                    sh '''
#!/bin/bash
set -e
npm install
npm run build
'''
                }
            }
        }

        /* =========================
           BACKEND – DOCKER
        ========================== */
        stage('Build & Push Backend Docker') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
#!/bin/bash
set -e

echo ">>> Docker build backend"
docker build -t ${BACKEND_IMAGE} ./movieApi

DOCKER_HOST=$(echo ${BACKEND_IMAGE} | cut -d/ -f1)

echo ">>> Docker login"
echo "$DOCKER_PASS" | docker login "$DOCKER_HOST" -u "$DOCKER_USER" --password-stdin

echo ">>> Docker push backend"
docker push ${BACKEND_IMAGE}

docker logout "$DOCKER_HOST"
'''
                }
            }
        }

        /* =========================
           FRONTEND – DOCKER
        ========================== */
        stage('Build & Push Frontend Docker') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
#!/bin/bash
set -e

echo ">>> Docker build frontend"
docker build -t ${FRONTEND_IMAGE} ./movieUi

DOCKER_HOST=$(echo ${FRONTEND_IMAGE} | cut -d/ -f1)

echo ">>> Docker login"
echo "$DOCKER_PASS" | docker login "$DOCKER_HOST" -u "$DOCKER_USER" --password-stdin

echo ">>> Docker push frontend"
docker push ${FRONTEND_IMAGE}

docker logout "$DOCKER_HOST"
'''
                }
            }
        }

        /* =========================
           DEPLOY
        ========================== */
        stage('Deploy with Docker Compose') {
            steps {
                dir("${DOCKER_COMPOSE_DIR}") {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
#!/bin/bash
set -e

# Connexion au registry privé
echo ">>> Docker login au registry privé"
echo "$DOCKER_PASS" | docker login 192.168.122.48:5001 --username "$DOCKER_USER" --password-stdin

# Pull des images backend/frontend et des services
echo ">>> Docker Compose pull"
docker compose pull

# Démarrage des services
echo ">>> Docker Compose up -d"
docker compose up -d

# Logout du registry privé
docker logout 192.168.122.48:5001
'''
                    }
                }
            }
        }
    }

    /* =========================
       POST
    ========================== */
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
