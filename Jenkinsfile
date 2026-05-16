pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "mohamedyassinebouneb/aston-villa-app"
        CONTAINER_NAME = "angular-app-deployed"
        HOST_PORT = "8085"
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Clonage du repository GitHub...'
                git branch: 'main', url: 'https://github.com/YassinoMed/TPEv.git'
            }
        }

        stage('Setup Tag') {
            steps {
                script {
                    env.DOCKER_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    echo "Docker Tag : ${env.DOCKER_TAG}"
                }
            }
        }

        stage('Build Image') {
            steps {
                echo "Construction de ${DOCKER_IMAGE}:${DOCKER_TAG}..."
                dir('datacamp_docker_angular') {
                    sh "docker build --pull --cache-from ${DOCKER_IMAGE}:latest -t ${DOCKER_IMAGE}:${DOCKER_TAG} -t ${DOCKER_IMAGE}:latest ."
                }
            }
        }

        stage('Test Docker Image') {
            steps {
                echo "Test de l'image (nginx -t)..."
                sh "docker run --rm ${DOCKER_IMAGE}:${DOCKER_TAG} nginx -t"
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKERHUB_PASS', usernameVariable: 'DOCKERHUB_USER')]) {
                    sh "echo \$DOCKERHUB_PASS | docker login -u \$DOCKERHUB_USER --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Deploy Container (SSH to jenkins-agent)') {
            steps {
                echo "Déploiement via SSH sur jenkins-agent..."
                sshagent(credentials: ['agent-key']) {
                    sh """
                        SSH_CMD="ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null jenkins@jenkins-agent"

                        echo "1. Pull de la nouvelle image..."
                        \$SSH_CMD "docker pull ${DOCKER_IMAGE}:${DOCKER_TAG}"

                        echo "2. Arrêt de l'ancien conteneur..."
                        \$SSH_CMD "docker stop ${CONTAINER_NAME} || true"
                        \$SSH_CMD "docker rm ${CONTAINER_NAME} || true"

                        echo "3. Lancement du nouveau conteneur (port ${HOST_PORT}:80)..."
                        \$SSH_CMD "docker run -d --name ${CONTAINER_NAME} -p ${HOST_PORT}:80 ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sleep time: 5, unit: 'SECONDS'
                sshagent(credentials: ['agent-key']) {
                    sh """
                        SSH_CMD="ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null jenkins@jenkins-agent"
                        APP_IP=\$(\$SSH_CMD "docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${CONTAINER_NAME}")
                        echo "IP du conteneur : \$APP_IP"
                        if \$SSH_CMD "curl -sf http://\${APP_IP}:80 > /dev/null"; then
                            echo "Déploiement OK — accessible sur http://localhost:${HOST_PORT}"
                        else
                            echo "Échec : l'application ne répond pas."
                            exit 1
                        fi
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline terminée avec succès. App dispo sur http://localhost:${HOST_PORT}"
            sh "docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true"
        }
        failure {
            echo "Pipeline échouée — voir les logs ci-dessus."
        }
        always {
            echo "Fin du job."
        }
    }
}
