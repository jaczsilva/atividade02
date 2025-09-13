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

    // Nomes de rede, containers e volume de dados do MySQL
    NETWORK       = "atividade02_net"
    WEB_CONTAINER = "atividade02_web_app"
    DB_CONTAINER  = "atividade02_db_mysql"
    VOLUME_NAME   = "atividade02_mysql_data"
  }

  stages {
    stage('Cleanup') {
      steps {
        sh '''
          echo "[CLEANUP] Removendo containers/imagens/volume antigos..."
          docker stop ${WEB_CONTAINER} || true
          docker rm   ${WEB_CONTAINER} || true

          docker stop ${DB_CONTAINER}  || true
          # remove container + volume anônimo ligado a ele (se houver)
          docker rm -v ${DB_CONTAINER} || true

          # remove volume nomeado (força reinit do MySQL no próximo run)
          docker volume rm ${VOLUME_NAME} || true

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
        sh 'echo "[DEBUG] Conteúdo de db/:" && ls -la db  || true'
      }
    }

    stage('Construção') {
      steps {
        sh '''
          echo "[BUILD] Criando rede..."
          docker network create ${NETWORK} || true

          echo "[BUILD] Build DB..."
          docker build -t ${DB_IMAGE}:latest -f ${DB_DOCKERFILE} ${DB_CONTEXT}

          echo "[BUILD] Build WEB..."
          docker build -t ${WEB_IMAGE}:latest -f ${WEB_DOCKERFILE} ${WEB_CONTEXT}
        '''
      }
    }

    stage('Entrega') {
      steps {
        sh '''
          set -e

          echo "[DELIVERY] Subindo banco (imagem custom) ..."
          docker run -d --name ${DB_CONTAINER} --network ${NETWORK} \
            -e MYSQL_ROOT_PASSWORD=root \
            -e MYSQL_DATABASE=docker_e_kubernetes \
            -v ${VOLUME_NAME}:/var/lib/mysql \
            ${DB_IMAGE}:latest \
            --character-set-server=utf8mb4 \
            --collation-server=utf8mb4_unicode_ci

          echo "[DELIVERY] Aguardando DB responder mysqladmin ping..."
          ATTEMPTS=30
          until docker exec ${DB_CONTAINER} sh -lc "mysqladmin ping -uroot -proot --silent" ; do
            ATTEMPTS=$((ATTEMPTS-1))
            if [ $ATTEMPTS -le 0 ]; then
              echo "[ERROR] MySQL não respondeu a tempo. Logs:"
              docker logs ${DB_CONTAINER} || true
              exit 1
            fi
            echo "[WAIT] MySQL ainda iniciando... aguardando 2s"
            sleep 2
          done
          echo "[OK] MySQL respondeu ao ping."

          echo "[CHECK] Conferindo se a base/tabela existem..."
          # Cria DB se não existir (idempotente)
          docker exec ${DB_CONTAINER} sh -lc "mysql -uroot -proot -e \\"CREATE DATABASE IF NOT EXISTS docker_e_kubernetes CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;\\""

          EXISTE=$(docker exec ${DB_CONTAINER} sh -lc "mysql -N -uroot -proot -e \\"USE docker_e_kubernetes; SHOW TABLES LIKE 'atividade02';\\"")
          if [ -z "$EXISTE" ]; then
            echo "[APPLY] Tabela não encontrada. Tentando aplicar schema do init..."
            if docker exec ${DB_CONTAINER} sh -lc "[ -f /docker-entrypoint-initdb.d/01_schema.sql ]" ; then
              docker exec ${DB_CONTAINER} sh -lc "mysql --default-character-set=utf8mb4 -uroot -proot < /docker-entrypoint-initdb.d/01_schema.sql"
            else
              echo "[FALLBACK] 01_schema.sql não está na imagem. Criando tabela mínima via SQL inline."
              docker exec ${DB_CONTAINER} sh -lc "mysql -uroot -proot -e \\
                \\"USE docker_e_kubernetes; \\
                CREATE TABLE IF NOT EXISTS atividade02 ( \\
                  ID INT PRIMARY KEY, \\
                  FirstName VARCHAR(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL, \\
                  LastName  VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL, \\
                  Age INT NULL, \\
                  Height DECIMAL(4,2) NULL \\
                ) CHARACTER SET=utf8mb4 COLLATE=utf8mb4_unicode_ci; \\
              \\""
            fi
          fi

          echo "[VERIFY] Tabelas atuais:"
          docker exec ${DB_CONTAINER} sh -lc "mysql -uroot -proot -e \\"USE docker_e_kubernetes; SHOW TABLES;\\""

          echo "[DELIVERY] Descobrindo IP do DB..."
          DB_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${DB_CONTAINER} || true)
          echo "[DELIVERY] DB_IP='${DB_IP}'"

          ADD_HOST_ARG=""
          if [ -n "${DB_IP}" ]; then
            ADD_HOST_ARG="--add-host ${DB_CONTAINER}:${DB_IP}"
          else
            echo "[WARN] DB_IP vazio. Prosseguindo sem --add-host (DNS da rede Docker)."
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

          echo "[INFO] Containers no ar:"
          docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
        '''
      }
    }
  }

  post {
    always {
      echo "[POST] Estado final dos containers/imagens:"
      sh 'docker ps -a || true'
      sh 'docker images || true'
      echo "[POST] Teste rápido de conectividade DB/tabela:"
      sh '''
        set +e
        docker exec ${DB_CONTAINER} sh -lc "mysql -uroot -proot -e \\"USE docker_e_kubernetes; SELECT COUNT(*) AS registros FROM atividade02;\\"" || true
      '''
    }
  }
}
