pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        NEXUS_MAVEN_REPO = "http://192.168.11.104:8081/repository/spring-web-app-hosted/"
        DOCKER_REPO = "192.168.11.104:5001"
        BACKEND_IMAGE = "${DOCKER_REPO}/backend-app:latest"
        FRONTEND_IMAGE = "${DOCKER_REPO}/frontend-app:latest"
        GIT_APP_REPO = "https://github.com/camou92/tpjenkins-spring.git"
        DOCKER_COMPOSE_DIR = "docker"
    }

    stages {

        stage("Clean workspace") {
            steps {
                cleanWs()
            }
        }

        stage('Checkout App Repo') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: "${GIT_APP_REPO}"]]])
            }
        }

        stage('Build Backend & Deploy Artifact') {
            steps {
                dir('backend') {
                    withCredentials([usernamePassword(credentialsId: 'nexus-cred',
                        usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        sh '''
                            mvn clean install
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
                            mvn deploy -s settings.xml -DskipTests
                        '''
                    }
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh '''
                        npm install
                        npm run build --prod
                    '''
                }
            }
        }

        stage('Build & Push Backend Docker') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-cred',
                    usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        docker build -t ${BACKEND_IMAGE} ./backend
                        DOCKER_HOST=$(echo ${BACKEND_IMAGE} | cut -d/ -f1)
                        echo "$DOCKER_PASS" | docker login $DOCKER_HOST --username "$DOCKER_USER" --password-stdin
                        docker push ${BACKEND_IMAGE}
                        docker logout $DOCKER_HOST
                    '''
                }
            }
        }

        stage('Build & Push Frontend Docker') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-cred',
                    usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        docker build -t ${FRONTEND_IMAGE} ./frontend
                        DOCKER_HOST=$(echo ${FRONTEND_IMAGE} | cut -d/ -f1)
                        echo "$DOCKER_PASS" | docker login $DOCKER_HOST --username "$DOCKER_USER" --password-stdin
                        docker push ${FRONTEND_IMAGE}
                        docker logout $DOCKER_HOST
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                dir("${DOCKER_COMPOSE_DIR}") {
                    sh '''
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
                message: "🎉 SUCCESS — Artifact et images backend & frontend déployés 🚀"
            )
        }
        failure {
            slackSend(
                channel: '#tous-camoutech',
                color: '#ff0000',
                message: "❌ Pipeline échoué ⚠️"
            )
        }
    }
}
