# Innovatech — Backend Despachos - Funcionamiento en rama DEPLOY

API REST del microservicio de **Despachos** de Innovatech Chile, construida con Spring Boot 3 y MySQL. Contenedorizada con Docker y desplegada automáticamente en AWS EC2 mediante GitHub Actions.

Forma parte del proyecto de **Evaluación Parcial N°2 — Introducción a Herramientas DevOps (ISY1101)**.

## Arquitectura general

```
┌────────────────────┐         ┌────────────────────┐         ┌────────────────────┐
│  EC2 Frontend       │  HTTPS │  EC2 Back-Despachos │  JDBC  │     AWS RDS         │
│  (pública, :80)     │ ─────► │  (privada, :8081)   │ ─────► │     MySQL           │
│  nginx + SPA        │        │  Spring Boot        │        │     despachos_db    │
└────────────────────┘        └────────────────────┘        └────────────────────┘
```

Este servicio vive en una EC2 **privada** y solo es alcanzable desde el Security Group del frontend. La persistencia se delega a un MySQL en **AWS RDS**.

## Estructura del repositorio

```
back-Despachos_SpringBoot/
├── Springboot-API-REST-DESPACHO/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/citt/
│   │   │   │   ├── config/                  # OpenAPI, CORS
│   │   │   │   ├── controller/              # DespachoController
│   │   │   │   ├── persistence/
│   │   │   │   │   ├── entity/              # Despacho.java
│   │   │   │   │   ├── repository/          # DespachoRepository
│   │   │   │   │   └── services/            # DespachoService + Impl
│   │   │   │   └── exceptions/              # Manejo global de errores
│   │   │   └── resources/
│   │   │       └── application.properties
│   │   └── test/
│   ├── pom.xml
│   ├── Dockerfile                           # Multi-stage maven → JRE
│   ├── .dockerignore
│   └── mvnw / mvnw.cmd
├── docker-compose.deploy.yml                # Compose de PRODUCCIÓN (EC2)
├── .github/
│   └── workflows/
│       └── deploy.yml                       # Pipeline CI/CD
└── README.md
```

---

## Ejecución local sin Docker

Requiere Java 17 y un MySQL local en `localhost:3306`.

```bash
cd Springboot-API-REST-DESPACHO

# Variables de entorno
export DB_ENDPOINT=localhost
export DB_PORT=3306
export DB_NAME=despachos_db
export DB_USERNAME=root
export DB_PASSWORD=tu_password

./mvnw spring-boot:run
```

Swagger UI: http://localhost:8081/swagger-ui.html

---

## Ejecución local con Docker

```bash
cd Springboot-API-REST-DESPACHO

docker build -t innovatech-back-despachos:dev .

docker run --rm -p 8081:8081 \
  -e DB_ENDPOINT=host.docker.internal \
  -e DB_PORT=3306 \
  -e DB_NAME=despachos_db \
  -e DB_USERNAME=root \
  -e DB_PASSWORD=tu_password \
  innovatech-back-despachos:dev
```

Para el **stack completo** (frontend + ambos backends + MySQL en contenedor), usar el `docker-compose.yml` de la raíz `proyecto semestral/`.

---

## Variables de entorno

| Variable | Descripción | Ejemplo |
|---|---|---|
| `DB_ENDPOINT` | Host del MySQL (RDS endpoint o nombre de servicio en compose) | `xxx.rds.amazonaws.com` |
| `DB_PORT` | Puerto MySQL | `3306` |
| `DB_NAME` | Nombre de la base de datos | `despachos_db` |
| `DB_USERNAME` | Usuario MySQL | `innovatech` |
| `DB_PASSWORD` | Contraseña MySQL | `xxxxxx` |
| `JAVA_OPTS` | Tuning de la JVM (opcional) | `-Xms256m -Xmx512m` |

La JDBC URL del [`application.properties`](./Springboot-API-REST-DESPACHO/src/main/resources/application.properties) incluye `createDatabaseIfNotExist=true`, por lo que Hibernate crea la base si no existe.

## Pipeline CI/CD (GitHub Actions)

Archivo: [.github/workflows/deploy.yml](./.github/workflows/deploy.yml)

**Flujo del pipeline:**

```
push → build (multi-stage) → push a Docker Hub → SSH a EC2 → pull + restart
```

**Pasos:**

1. **Checkout** del código fuente.
2. **Login a Docker Hub** con `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN`.
3. **Setup Buildx** para usar caché remota de capas (GitHub Actions cache).
4. **Build & Push** con dos tags:
   - `latest` — móvil
   - `${{ github.sha }}` — inmutable, para trazabilidad y rollback
5. **SSH a la EC2 Back-Despachos** y ejecuta:
   ```bash
   docker compose -f docker-compose.deploy.yml pull
   docker compose -f docker-compose.deploy.yml up -d --remove-orphans
   docker image prune -f
   ```

### Secrets necesarios

| Secret | Descripción |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub |
| `DOCKERHUB_TOKEN` | Access Token (no la contraseña) |
| `EC2_DESPACHOS_HOST` | IP/DNS de la EC2 de Despachos |
| `EC2_USER` | `ubuntu` o `ec2-user` |
| `EC2_SSH_KEY` | Contenido del `.pem` |
| `DB_ENDPOINT` | Endpoint del RDS |
| `DB_PORT` | `3306` |
| `DB_NAME_DESPACHOS` | `despachos_db` |
| `DB_USERNAME` | Usuario RDS |
| `DB_PASSWORD` | Password RDS |

---

## Despliegue en AWS EC2

### 1. Preparar la EC2 (una sola vez)

```bash
sudo apt-get update -y
sudo apt-get install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker

mkdir -p ~/innovatech-back-despachos
cd ~/innovatech-back-despachos
# Subir docker-compose.deploy.yml o clonar este repo
```

### 2. Security Group recomendado

| Puerto | Origen | Justificación |
|---|---|---|
| 8081 | Security Group del frontend | Solo el front puede consumir la API |
| 22 | Tu IP fija | SSH administrativo |

### 3. RDS

| Configuración | Valor |
|---|---|
| Motor | MySQL 8 |
| Security Group | Permitir 3306 desde SG de los backends |
| Acceso público | **No** |

### 4. Disparar el despliegue

```bash
git checkout -b deploy
git push origin deploy
```

## Información del curso

| | |
|---|---|
| Asignatura | ISY1101 — Introducción a Herramientas DevOps |
| Evaluación | EP2 (30%) |
| Institución | Duoc UC |
| Año | 2025 |
