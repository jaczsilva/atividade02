pipeline {
    agent any

    environment {
    BASE_DIR = "."
    WEB_IMAGE = "atividade02-web"
    DB_IMAGE  = "atividade02-db"

    // Dockerfiles
    WEB_DOCKERFILE = "Dockerfile.web"
    DB_DOCKERFILE  = "Dockerfile.mysql"

    // Contextos (pastas) de build
    WEB_CONTEXT = "web"
    DB_CONTEXT  = "db"

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
                sh 'pwd && ls -la'
                sh 'echo "[DEBUG] Conteúdo da raiz:" && ls -la ${BASE_DIR}'
                sh 'echo "[DEBUG] Conteúdo de web/:" && ls -la web || true'
                sh 'echo "[DEBUG] Conteúdo de db/:" && ls -la db || true'
            }
        }

        stage('Construção') {
  steps {
    sh '''
      echo "[BUILD] Criando rede..."
      docker network create ${NETWORK} || true

      echo "[BUILD] Build das imagens (contextos dedicados)..."
      # DB: Dockerfile fica em db/Dockerfile.mysql e o contexto é a pasta db/
      docker build -t ${DB_IMAGE}:latest  -f db/Dockerfile.mysql  db

      # WEB: Dockerfile fica em web/Dockerfile.web e o contexto é a pasta web/
      docker build -t ${WEB_IMAGE}:latest -f web/Dockerfile.web web
    '''
  }
}

        stage('Entrega') {
            steps {
                sh '''
                    echo "[DELIVERY] Subindo banco..."
                    docker run -d --name ${DB_CONTAINER} --network ${NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=root \
                        -e MYSQL_DATABASE=atividade02 \
                        ${DB_IMAGE}:latest

                    echo "[DELIVERY] Aguardando DB subir..."
                    sleep 25

                    echo "[DELIVERY] Subindo aplicação web..."
                    docker run -d --name ${WEB_CONTAINER} --network ${NETWORK} \
                    -p 5000:5000 \
                    \
                    # Variáveis no padrão DB_* (se seu main.py usar)
                    -e DB_HOST=${DB_CONTAINER} \
                    -e DB_NAME=atividade02 \
                    -e DB_USER=root \
                    -e DB_PASS=root \
                    \
                    # Variáveis no padrão MYSQL_* (cobrir todos os nomes comuns)
                    -e MYSQL_ADDRESS=${DB_CONTAINER} \
                    -e MYSQL_HOST=${DB_CONTAINER} \
                    -e MYSQL_SERVER=${DB_CONTAINER} \
                    -e MYSQL_PORT=3306 \
                    -e MYSQL_DATABASE=atividade02 \
                    -e MYSQL_USERNAME=root \
                    -e MYSQL_PASSWORD=root \
                    ${WEB_IMAGE}:latest \
                '''
            }
        }
    }
}
