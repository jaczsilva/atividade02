pipeline {
    agent any

    environment {
        WEB_IMAGE = "atividade02-web"
        DB_IMAGE  = "atividade02-db"

        WEB_DOCKERFILE = "web/Dockerfile.web"
        DB_DOCKERFILE  = "db/Dockerfile.mysql"

        WEB_CONTAINER = "atividade02_web_app"
        DB_CONTAINER  = "atividade02_db_mysql"
        NETWORK = "atividade02_net"
    }

    stages {
        stage('Cleanup') {
            steps {
                sh '''
                    echo "[CLEANUP] Removendo containers e imagens antigas..."
                    docker stop ${WEB_CONTAINER} || true
                    docker rm ${WEB_CONTAINER} || true
                    docker stop ${DB_CONTAINER} || true
                    docker rm ${DB_CONTAINER} || true
                    docker network rm ${NETWORK} || true
                    docker rmi ${WEB_IMAGE}:latest || true
                    docker rmi ${DB_IMAGE}:latest || true
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Construção') {
            steps {
                sh '''
                    echo "[BUILD] Criando rede..."
                    docker network create ${NETWORK} || true

                    echo "[BUILD] Build das imagens..."
                    docker build -t ${DB_IMAGE}:latest -f ${DB_DOCKERFILE} .
                    docker build -t ${WEB_IMAGE}:latest -f ${WEB_DOCKERFILE} .
                '''
            }
        }

        stage('Entrega') {
            steps {
                sh '''
                    echo "[DELIVERY] Subindo banco..."
                    docker run -d --name ${DB_CONTAINER} --network ${NETWORK}                         -e MYSQL_ROOT_PASSWORD=root                         -e MYSQL_DATABASE=atividade02                         ${DB_IMAGE}:latest

                    echo "[DELIVERY] Subindo aplicação web..."
                    docker run -d --name ${WEB_CONTAINER} --network ${NETWORK}                         -p 5000:5000                         -e DB_HOST=${DB_CONTAINER}                         -e DB_NAME=atividade02                         -e DB_USER=root                         -e DB_PASS=root                         ${WEB_IMAGE}:latest
                '''
            }
        }
    }
}