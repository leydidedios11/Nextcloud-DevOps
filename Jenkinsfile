pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        PROJECT_NAME = "nextcloud-ci-${BUILD_NUMBER}"

        MARIADB_ROOT_PASSWORD = credentials('nextcloud-db-root-password')
        MARIADB_PASSWORD = credentials('nextcloud-db-password')
        NEXTCLOUD_ADMIN_PASSWORD = credentials('nextcloud-admin-password')

        MARIADB_DATABASE = 'nextcloud'
        MARIADB_USER = 'nextcloud'
        NEXTCLOUD_ADMIN_USER = 'admin'
        NEXTCLOUD_PORT = '18080'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Descargando el codigo fuente desde GitHub'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Construyendo las imagenes personalizadas'

                sh '''
                    set -eu
                    docker version
                    docker compose version
                    docker compose -p "$PROJECT_NAME" build db app
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Iniciando los servicios para ejecutar las pruebas'

                sh '''
                    set -eu

                    docker compose -p "$PROJECT_NAME" up -d

                    echo "Esperando que MariaDB este disponible..."

                    DB_READY=0

                    for i in $(seq 1 30); do
                        if docker compose -p "$PROJECT_NAME" exec -T db \
                            healthcheck.sh --connect --innodb_initialized; then
                            DB_READY=1
                            break
                        fi

                        sleep 5
                    done

                    if [ "$DB_READY" -ne 1 ]; then
                        echo "ERROR: MariaDB no alcanzo el estado saludable."
                        docker compose -p "$PROJECT_NAME" logs db
                        exit 1
                    fi

                    echo "MariaDB esta saludable."

                    echo "Comprobando comunicacion entre Nextcloud y MariaDB..."

                    docker compose -p "$PROJECT_NAME" exec -T app php -r '
                    $db = new mysqli(
                        getenv("MYSQL_HOST"),
                        getenv("MYSQL_USER"),
                        getenv("MYSQL_PASSWORD"),
                        getenv("MYSQL_DATABASE")
                    );

                    if ($db->connect_errno) {
                        fwrite(STDERR, $db->connect_error . PHP_EOL);
                        exit(1);
                    }

                    echo "Conexion entre Nextcloud y MariaDB exitosa." . PHP_EOL;
                    $db->close();
                    '

                    echo "Esperando la inicializacion de Nextcloud..."

                    APP_READY=0

                    for i in $(seq 1 36); do
                        if docker compose -p "$PROJECT_NAME" exec -T app php occ status; then
                            APP_READY=1
                            break
                        fi

                        sleep 5
                    done

                    if [ "$APP_READY" -ne 1 ]; then
                        echo "ERROR: Nextcloud no completo su inicializacion."
                        docker compose -p "$PROJECT_NAME" logs app
                        exit 1
                    fi

                    echo "Pruebas completadas correctamente."
                '''
            }
        }
    }

    post {
        always {
            echo 'Limpiando los recursos temporales de esta ejecucion'

            sh '''
                docker compose -p "$PROJECT_NAME" down --volumes --remove-orphans || true
            '''
        }

        success {
            echo 'Pipeline finalizado correctamente.'
        }

        failure {
            echo 'Pipeline finalizado con errores. Revisar la consola.'
        }
    }
}