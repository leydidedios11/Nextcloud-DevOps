# Nextcloud DevOps - Integracion Continua

## 1. Descripcion

Proyecto academico de integracion continua basado en Nextcloud y MariaDB,
utilizando Docker, Docker Compose, GitHub y GitHub Actions.

## 2. Arquitectura

- Nextcloud: aplicacion web de almacenamiento y colaboracion.
- MariaDB: servicio de base de datos relacional.
- Docker Compose: orquestacion de contenedores.
- Docker Network: comunicacion privada entre servicios.
- GitHub Actions: integracion continua y construccion automatizada.

## 3. Requisitos

- Docker Desktop
- Docker Compose v2
- Git
- Windows 10/11 o Linux

## 4. Configuracion

Copiar el archivo de ejemplo:

```powershell
Copy-Item .env.example .env