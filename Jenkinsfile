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
      set -e

      echo "[DELIVERY] Subindo banco..."
      docker run -d --name ${DB_CONTAINER} --network ${NETWORK} \
        -e MYSQL_ROOT_PASSWORD=root \
        -e MYSQL_DATABASE=docker_e_kubernetes \
        ${DB_IMAGE}:latest

      echo "[DELIVERY] Aguardando DB responder mysqladmin ping..."
      # Tenta por até ~60s (12 x 5s). Se falhar, mostra logs e aborta.
      ATTEMPTS=12
      until docker exec ${DB_CONTAINER} sh -c "mysqladmin ping -uroot -proot --silent" ; do
        ATTEMPTS=$((ATTEMPTS-1))
        if [ $ATTEMPTS -le 0 ]; then
          echo "[ERROR] MySQL não respondeu a tempo. Logs:"
          docker logs ${DB_CONTAINER} || true
          exit 1
        fi
        echo "[WAIT] MySQL ainda iniciando... aguardando 5s"
        sleep 5
      done
      echo "[OK] MySQL respondeu ao ping."

      echo "[DELIVERY] Descobrindo IP do DB..."
      DB_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${DB_CONTAINER} || true)
      echo "[DELIVERY] DB_IP='${DB_IP}'"

      ADD_HOST_ARG=""
      if [ -n "${DB_IP}" ]; then
        ADD_HOST_ARG="--add-host ${DB_CONTAINER}:${DB_IP}"
      else
        echo "[WARN] DB_IP vazio. Prosseguindo sem --add-host (confiaremos no DNS da rede Docker)."
      fi

      echo "[DELIVERY] Subindo aplicação web..."
      docker run -d --name ${WEB_CONTAINER} --network ${NETWORK} ${ADD_HOST_ARG} -p 5000:5000 \
        -e DB_HOST=${DB_CONTAINER} \
        -e DB_NAME=docker_e_kubernetes \
        -e DB_USER=root \
        -e DB_PASS=root \
        -e MYSQL_ADDRESS=${DB_CONTAINER} \
        -e MYSQL_HOST=${DB_CONTAINER} \
        -e MYSQL_SERVER=${DB_CONTAINER} \
        -e MYSQL_ADDR=${DB_IP} \
        -e MYSQL_PORT=3306 \
        -e MYSQL_DATABASE=docker_e_kubernetes \
        -e MYSQL_DBNAME=docker_e_kubernetes \
        -e MYSQL_DB=docker_e_kubernetes \
        -e MYSQL_USERNAME=root \
        -e MYSQL_USER=root \
        -e MYSQL_PASSWORD=root \
        -e MYSQL_PASS=root \
        ${WEB_IMAGE}:latest
    '''
  }
}
