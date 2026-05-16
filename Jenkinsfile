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

        stage('Deploy Container') {
            steps {
                echo "Déploiement local du conteneur..."
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} -p ${HOST_PORT}:80 ${DOCKER_IMAGE}:${DOCKER_TAG}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sleep time: 5, unit: 'SECONDS'
                sh """
                    APP_IP=\$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${CONTAINER_NAME})
                    if curl -sf http://\${APP_IP}:80 > /dev/null; then
                        echo "Déploiement OK — accessible sur http://localhost:${HOST_PORT}"
                    else
                        echo "Échec : l'application ne répond pas."
                        exit 1
                    fi
                """
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
