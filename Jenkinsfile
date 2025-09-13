pipeline {
  agent any

  environment {
    // Imagens
    WEB_IMAGE = "atividade02-web"
    DB_IMAGE  = "atividade02-db"

    // Dockerfiles (caminhos no repo)
    WEB_DOCKERFILE = "web/Dockerfile.web"
    DB_DOCKERFILE  = "db/Dockerfile.mysql"

    // Contextos de build
    WEB_CONTEXT = "web"
    DB_CONTEXT  = "db"

    // Nomes de rede e containers
    NETWORK      = "atividade02_net"
    WEB_CONTAINER= "atividade02_web_app"
    DB_CONTAINER = "atividade02_db_mysql"
  }

  stages {
    stage('Cleanup') {
      steps {
        sh '''
          echo "[CLEANUP] Removendo containers/imagens antigos..."
          docker stop ${WEB_CONTAINER} || true
          docker rm   ${WEB_CONTAINER} || true
          docker stop ${DB_CONTAINER} || true
          docker rm   ${DB_CONTAINER} || true
          docker network rm ${NETWORK} || true
          docker rmi ${WEB_IMAGE}:latest || true
          docker rmi ${DB_IMAGE}:latest  || true
        '''
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
        sh 'pwd && ls -la'
        sh 'echo "[DEBUG] Conteúdo de web/:" && ls -la web || true'
        sh 'echo "[DEBUG] Conteúdo de db/:"  && ls -la db  || true'
      }
    }

    stage('Construção') {
      steps {
        sh '''
          echo "[BUILD] Criando rede..."
          docker network create ${NETWORK} || true

          echo "[BUILD] Build DB..."
          docker build -t ${DB_IMAGE}:latest  -f ${DB_DOCKERFILE}  ${DB_CONTEXT}

          echo "[BUILD] Build WEB..."
          docker build -t ${WEB_IMAGE}:latest -f ${WEB_DOCKERFILE} ${WEB_CONTEXT}
        '''
      }
    }

    stage('Entrega') {
      steps {
        sh '''
          echo "[DELIVERY] Subindo banco..."
          docker run -d --name ${DB_CONTAINER} --network ${NETWORK} \
            -e MYSQL_ROOT_PASSWORD=root \
            -e MYSQL_DATABASE=docker_e_kubernetes \
            ${DB_IMAGE}:latest

          echo "[DELIVERY] Aguardando DB subir..."
          sleep 30

          echo "[DELIVERY] Descobrindo IP do DB..."
          DB_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${DB_CONTAINER})
          echo "[DELIVERY] DB_IP=${DB_IP}"

          echo "[DELIVERY] Subindo aplicação web..."
          # One-liner para não perder a imagem no final
          docker run -d --name ${WEB_CONTAINER} --network ${NETWORK} --add-host ${DB_CONTAINER}:${DB_IP} -p 5000:5000 -e DB_HOST=${DB_CONTAINER} -e DB_NAME=docker_e_kubernetes -e DB_USER=root -e DB_PASS=root -e MYSQL_ADDRESS=${DB_CONTAINER} -e MYSQL_HOST=${DB_CONTAINER} -e MYSQL_SERVER=${DB_CONTAINER} -e MYSQL_ADDR=${DB_IP} -e MYSQL_PORT=3306 -e MYSQL_DATABASE=docker_e_kubernetes -e MYSQL_DBNAME=docker_e_kubernetes -e MYSQL_DB=docker_e_kubernetes -e MYSQL_USERNAME=root -e MYSQL_USER=root -e MYSQL_PASSWORD=root -e MYSQL_PASS=root ${WEB_IMAGE}:latest
        '''
      }
    }
  }

  post {
    always {
      sh '''
        echo "[POST] docker ps -a:"
        docker ps -a || true
        echo "[POST] Logs web (tail 50):"
        docker logs --tail 50 ${WEB_CONTAINER} || true
      '''
    }
  }
}
