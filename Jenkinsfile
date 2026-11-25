
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
                echo 'Construyendo las imagenes personalizadas de Nextcloud y MariaDB'

                sh '''
                    set +x
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
                    set +x
                    set -eu

                    echo "Iniciando MariaDB y Nextcloud..."

                    docker compose -p "$PROJECT_NAME" up -d

                    echo "Esperando que MariaDB este disponible..."

                    DB_READY=0

                    for i in $(seq 1 30); do

                        if docker compose -p "$PROJECT_NAME" exec -T db \
                            healthcheck.sh --connect --innodb_initialized; then

                            DB_READY=1
                            break
                        fi

                        echo "MariaDB aun no esta lista. Intento $i de 30."
                        sleep 5
                    done

                    if [ "$DB_READY" -ne 1 ]; then
                        echo "ERROR: MariaDB no alcanzo el estado saludable."
                        docker compose -p "$PROJECT_NAME" logs db
                        exit 1
                    fi

                    echo "PRUEBA 1 EXITOSA: MariaDB esta saludable."

                    echo "Comprobando el controlador PDO MySQL de PHP..."

                    docker compose -p "$PROJECT_NAME" exec -T app php -r '
                    if (!extension_loaded("pdo_mysql")) {
                        fwrite(STDERR, "ERROR: El controlador PDO MySQL no esta disponible." . PHP_EOL);
                        exit(1);
                    }

                    echo "Controlador PDO MySQL disponible." . PHP_EOL;
                    '

                    echo "Comprobando comunicacion entre Nextcloud y MariaDB..."

                    docker compose -p "$PROJECT_NAME" exec -T app php -r '
                    $host = getenv("MYSQL_HOST");
                    $database = getenv("MYSQL_DATABASE");
                    $user = getenv("MYSQL_USER");
                    $password = getenv("MYSQL_PASSWORD");

                    $dsn = "mysql:host=" . $host .
                           ";dbname=" . $database .
                           ";charset=utf8mb4";

                    try {
                        $db = new PDO(
                            $dsn,
                            $user,
                            $password,
                            [
                                PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                                PDO::ATTR_TIMEOUT => 10
                            ]
                        );

                        $result = $db->query("SELECT 1")->fetchColumn();

                        if ((int) $result !== 1) {
                            throw new RuntimeException("La consulta de prueba no devolvio el resultado esperado.");
                        }

                        echo "PRUEBA 2 EXITOSA: Conexion entre Nextcloud y MariaDB mediante el servicio db." . PHP_EOL;

                        $db = null;

                    } catch (Throwable $e) {
                        fwrite(STDERR, "ERROR en la conexion con MariaDB: " . $e->getMessage() . PHP_EOL);
                        exit(1);
                    }
                    '

                    echo "Esperando la inicializacion de Nextcloud..."

                    APP_READY=0

                    for i in $(seq 1 36); do

                        if docker compose -p "$PROJECT_NAME" exec -T app \
                            php occ status | grep -q "installed: true"; then

                            APP_READY=1
                            break
                        fi

                        echo "Nextcloud aun no esta instalado. Intento $i de 36."
                        sleep 5
                    done

                    if [ "$APP_READY" -ne 1 ]; then
                        echo "ERROR: Nextcloud no completo su inicializacion."
                        docker compose -p "$PROJECT_NAME" logs app
                        exit 1
                    fi

                    echo "PRUEBA 3 EXITOSA: Nextcloud esta instalado y operativo."

                    echo "===== ESTADO FINAL DE LOS SERVICIOS ====="

                    docker compose -p "$PROJECT_NAME" ps

                    echo "TODAS LAS PRUEBAS FINALIZARON CORRECTAMENTE."
                '''
            }
        }
    }

    post {
        always {
            echo 'Limpiando los recursos temporales de esta ejecucion'

            sh '''
                set +x

                if ! docker compose -p "$PROJECT_NAME" down --volumes --remove-orphans; then
                    echo "ADVERTENCIA: La limpieza de recursos reporto un problema."
                fi
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
