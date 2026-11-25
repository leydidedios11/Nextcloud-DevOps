# Nextcloud con Docker Compose

¡Bienvenido! Este proyecto levanta Nextcloud y MariaDB en dos contenedores
conectados por una red privada de Docker. No hace falta instalar Nextcloud ni
MariaDB directamente en tu computador: Docker se encarga de preparar los
servicios.

## Qué vas a poner en marcha

- **Nextcloud (`app`)**: la aplicación web. La imagen se construye desde el
  Dockerfile del proyecto y escucha en el puerto 80 dentro del contenedor.
- **MariaDB (`db`)**: la base de datos de Nextcloud. Sus datos se guardan en un
  volumen para que no se pierdan al detener los contenedores.
- **Docker Compose**: inicia ambos servicios y los conecta a la red
  `nextcloud_network`. Nextcloud encuentra la base de datos usando el nombre
  del servicio `db`, no `localhost` ni una dirección IP.

## Antes de empezar

Necesitas:

1. Docker instalado y activo. En Windows o macOS puedes instalar
   [Docker Desktop](https://docs.docker.com/get-started/get-docker/); en Linux,
   instala [Docker Engine](https://docs.docker.com/engine/install/) y el plugin
   de Docker Compose.
2. Docker Compose v2, que se usa como `docker compose` con un espacio.
3. Git para clonar el repositorio.

Abre PowerShell, Terminal o una consola de Linux y comprueba que Docker responde:

```console
docker --version
docker compose version
```

Si alguno de esos comandos falla, abre Docker Desktop o instala Docker antes de
continuar.

## Descargar y configurar el proyecto

Clona el repositorio y entra en su carpeta:

```console
git clone https://github.com/leydidedios11/Nextcloud-DevOps.git
cd Nextcloud-DevOps
```

Crea tu archivo local de configuración a partir del ejemplo.

**Windows PowerShell:**

```powershell
Copy-Item .env.example .env
notepad .env
```

**macOS o Linux:**

```sh
cp .env.example .env
```

Abre `.env` con un editor de texto y cambia todos los valores
`CAMBIAR_POR_...` por contraseñas largas, únicas y distintas para MariaDB y para
el administrador de Nextcloud. No agregues comillas. Guarda el archivo antes
de seguir. El usuario administrador inicial es `admin`; puedes cambiarlo en
`NEXTCLOUD_ADMIN_USER`.

El archivo `.env` contiene contraseñas y no se debe subir a GitHub. Está excluido
por `.gitignore` y `.dockerignore`; el archivo seguro para compartir es
`.env.example`.

## Iniciar los contenedores

Desde la carpeta del proyecto, ejecuta:

```console
docker compose config --quiet
docker compose build app
docker compose up -d
```

El primer comando comprueba la configuración, el segundo construye la imagen de
Nextcloud y el tercero inicia los dos servicios en segundo plano. La primera
construcción puede tardar unos minutos porque Docker descarga las imágenes
necesarias.

Comprueba que ambos contenedores estén activos:

```console
docker compose ps
```

Cuando termine la instalación inicial, abre
<http://localhost:8080> en el navegador. Inicia sesión con el usuario y la
contraseña de administrador que configuraste en `.env`. Si cambiaste
`NEXTCLOUD_PORT`, usa ese puerto en la dirección.

## Comprobar la comunicación entre contenedores

En `compose.yaml`, Nextcloud tiene configurado `MYSQL_HOST=db`. Ese `db` es el
nombre del servicio de MariaDB dentro de la red de Docker. Comprueba primero que
Nextcloud terminó su instalación:

```console
docker compose exec -u www-data app php occ status
```

Para demostrar una petición HTTP real, usa WebDAV para guardar un archivo en
Nextcloud y descargarlo de nuevo. Estos comandos de PowerShell usan `curl.exe`
(no el alias `curl` de Windows PowerShell):

```powershell
Set-Content -NoNewline -Path .\prueba.txt -Value "conexion-ok"
curl.exe --fail --show-error --write-out "HTTP %{http_code}`n" --user admin --upload-file .\prueba.txt http://localhost:8080/remote.php/dav/files/admin/prueba.txt
curl.exe --fail --show-error --write-out "HTTP %{http_code}`n" --user admin http://localhost:8080/remote.php/dav/files/admin/prueba.txt --output .\prueba-recuperada.txt
Get-Content .\prueba-recuperada.txt
```

`curl.exe` pedirá la contraseña sin incluirla en el comando. Si cambiaste
`NEXTCLOUD_ADMIN_USER`, reemplaza `admin` tanto en `--user` como en la URL.
`Get-Content` debe mostrar `conexion-ok`. La solicitud llega a Nextcloud; la
aplicación guarda el archivo en su volumen y sus metadatos en MariaDB mediante
el servicio `db`. MariaDB usa el protocolo SQL, no HTTP, por lo que `curl` no se
conecta directamente a ese contenedor.

Como comprobación adicional de la comunicación con la base de datos, guarda un
valor de configuración de la aplicación y léelo de nuevo:

```console
docker compose exec -u www-data app php occ config:app:set core docker_demo --value=conexion-ok
docker compose exec -u www-data app php occ config:app:get core docker_demo
```

El segundo comando debe mostrar `conexion-ok`. `occ` se ejecuta dentro del
contenedor de Nextcloud y al guardar y leer el ajuste la aplicación consulta la
base de datos configurada en `db`.

## Comandos útiles

Ver los contenedores y sus puertos:

```console
docker compose ps
```

Ver los registros de ambos servicios (pulsa `Ctrl+C` para salir sin detenerlos):

```console
docker compose logs -f app db
```

Ver la red y los contenedores conectados:

```console
docker network inspect nextcloud_network
```

Detener los servicios sin borrar tus archivos ni la base de datos:

```console
docker compose down
```

Volver a iniciarlos después:

```console
docker compose up -d
```

> **Atención:** `docker compose down -v` también elimina los volúmenes con los
> archivos de Nextcloud y la base de datos. No lo ejecutes si quieres conservar
> tus datos.

## Si algo no funciona

- **Docker no responde:** abre Docker Desktop y espera a que indique que está
  iniciado; luego vuelve a ejecutar el comando.
- **El puerto 8080 está ocupado:** cambia `NEXTCLOUD_PORT` en `.env` (por
  ejemplo, a `8081`) y vuelve a ejecutar `docker compose up -d`.
- **Nextcloud aún no abre:** la primera instalación puede tardar. Revisa
  `docker compose ps` y los registros con `docker compose logs -f app db`.
- **Cambiaste una contraseña pero no se refleja:** las credenciales de MariaDB
  se usan al crear por primera vez el volumen. Cambiarlas en `.env` no cambia
  automáticamente una base de datos ya inicializada.

## Estructura del repositorio

```text
.
├── .github/workflows/docker-ci.yml  # Construcción y validación en GitHub Actions
├── docker/nextcloud/Dockerfile      # Imagen de la aplicación
├── compose.yaml                     # Servicios, red, puertos y volúmenes
├── .env.example                     # Plantilla de configuración, sin secretos
├── .dockerignore                    # Archivos que no entran al contexto de build
└── README.md                        # Esta guía
```

## Integración continua

GitHub Actions valida la configuración de Docker Compose y construye la imagen
de Nextcloud en cada push y pull request. Puedes revisar el resultado en la
pestaña **Actions** del repositorio.

## Evidencias para la entrega

El PDF de evidencias es un documento aparte y todavía debe prepararse. Incluye
capturas propias de:

1. `docker compose build app` terminado correctamente.
2. `docker compose up -d` y `docker compose ps` con los dos servicios activos.
3. `docker network inspect nextcloud_network`, mostrando ambos contenedores.
4. La prueba anterior con `occ`, donde se guarda y luego se obtiene
   `conexion-ok`.
5. Un diagrama sencillo con Nextcloud y MariaDB dentro de la misma red.
6. Una conclusión breve y referencias en formato APA.

Revisa que las capturas sean legibles y que el PDF incluya el enlace al
repositorio de GitHub.

Actualizacion de documentacion para validar la ejecucion automatica del Pipeline mediante Poll SCM.
