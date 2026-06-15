# Innovatech Chile — Sistema de Microservicios EP2

## 1. Descripción General

Plataforma de gestión operacional para Innovatech Chile compuesta por un monorepo con tres componentes desacoplados: un frontend web basado en React/Vite, y dos microservicios backend independientes construidos con Spring Boot 3.4.x sobre Java 17. El microservicio **Despachos** (puerto 8081) gestiona la logística de envíos; el microservicio **Ventas** (puerto 8082) gestiona el ciclo de transacciones comerciales. Ambos persisten en una base de datos MySQL 8.0 compartida. El sistema está completamente contenedorizado con Docker y se despliega de forma automatizada sobre AWS EC2 mediante un pipeline CI/CD con GitHub Actions.

---

## 2. Arquitectura de la Solución

```
                         ┌─────────────────────────────────────────┐
                         │         network_innovatech (bridge)      │
                         │                                          │
  Usuario  ──HTTP:80──►  │  [Frontend Nginx]                        │
                         │       │                                  │
                         │       ├──HTTP:8081──► [Backend Despachos]│
                         │       │                    │             │
                         │       └──HTTP:8082──► [Backend Ventas]   │
                         │                            │             │
                         │                      [MySQL 8.0 :3306]  │
                         └─────────────────────────────────────────┘
```

**Separación de responsabilidades:**

El frontend (React 18 + Vite 5 + Tailwind CSS) es una SPA que se sirve como contenido estático desde Nginx. No contiene lógica de negocio: consume las APIs REST expuestas por cada microservicio backend via HTTP. Esta separación garantiza que el frontend sea stateless y escalable de forma independiente.

**Stack tecnológico por componente:**

| Componente | Runtime | Puerto interno | Gestor de paquetes |
|---|---|---|---|
| Frontend | Node 22 / Nginx 1.25 | 80 | pnpm 9.15.4 |
| Backend Despachos | JRE 17 Alpine | 8081 | Maven 3.9 |
| Backend Ventas | JRE 17 Alpine | 8080 (→8082 externo) | Maven 3.9 |
| Base de Datos | MySQL 8.0 | 3306 | — |

**Configuración de conexión a base de datos (`application.properties`):**

Los backends no hardcodean credenciales. La cadena de conexión se construye desde variables de entorno inyectadas en tiempo de ejecución por Docker Compose:

```properties
spring.datasource.url=jdbc:mysql://${DB_ENDPOINT}:${DB_PORT}/${DB_NAME}?useSSL=false&serverTimezone=UTC
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=update
```

Esto permite reutilizar la misma imagen Docker en entornos locales (apuntando al contenedor `mysql-db`) y en producción (apuntando a AWS RDS) sin reconstruir la imagen.

**API REST documentada:** El `pom.xml` incluye la dependencia `springdoc-openapi-starter-webmvc-ui:2.7.0`, lo que expone automáticamente Swagger UI en `/swagger-ui.html` de cada microservicio, facilitando la integración y validación de contratos de API.

---

## 3. Contenedorización — Justificación IE1

### 3.1 Multi-Stage Build

Todos los Dockerfiles (Frontend y ambos Backends) implementan el patrón **multi-stage build**, que es la práctica central de optimización de imágenes en Docker.

**Backends Spring Boot (Despachos y Ventas):**

```dockerfile
# ETAPA 1: BUILD — imagen completa de Maven con JDK (>500 MB)
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B   # Cache de dependencias como capa separada
COPY src ./src
RUN mvn clean package -DskipTests

# ETAPA 2: RUNTIME — solo JRE Alpine (~80 MB)
FROM eclipse-temurin:17-jre-alpine
COPY --from=builder /app/target/*.jar app.jar
```

La imagen final **no contiene Maven, el JDK, el código fuente ni los directorios `target/`**. Solo transporta el JAR ejecutable sobre un JRE mínimo. La reducción de superficie de ataque es directamente proporcional a la reducción del tamaño de imagen.

La instrucción `RUN mvn dependency:go-offline -B` antes de copiar el código fuente es una optimización de caché de capas: Docker reutiliza esta capa en rebuilds sucesivos si `pom.xml` no cambia, acelerando el ciclo de desarrollo.

**Frontend React/Vite:**

```dockerfile
# ETAPA 1: BUILD — Node 22 + pnpm para compilar la SPA
FROM node:22-alpine AS builder
RUN corepack enable && corepack prepare pnpm@9.15.4 --activate
COPY package.json pnpm-lock.yaml* ./
RUN pnpm install --frozen-lockfile     # Reproducibilidad: lockfile obligatorio
COPY . .
RUN npx vite build                     # Genera /dist con assets estáticos

# ETAPA 2: RUNTIME — Nginx Alpine sirve HTML/CSS/JS estáticos
FROM nginx:1.25-alpine
COPY --from=builder /app/front_despacho-main/dist /usr/share/nginx/html
```

`node_modules` (que puede superar 300 MB) queda completamente excluido de la imagen de producción. Nginx sirve únicamente los archivos compilados en `/dist`.

La instrucción `pnpm config set verify-store-integrity true` habilita verificación criptográfica de dependencias contra el store de pnpm, mitigando ataques de supply-chain (dependencias comprometidas).

### 3.2 Usuario No-Root (Seguridad)

Los tres Dockerfiles crean usuarios sin privilegios y los activan antes de la instrucción `ENTRYPOINT`/`CMD`:

**Backends:**
```dockerfile
RUN addgroup -S springboot && adduser -S springboot -G springboot
RUN mkdir /app/logs && chown -R springboot:springboot /app
USER springboot
```

**Frontend:**
```dockerfile
RUN chown -R nginx:nginx /var/run/nginx.pid /var/cache/nginx /var/log/nginx /usr/share/nginx/html
USER nginx
```

Ejecutar procesos como root dentro de un contenedor equivale a ejecutarlos como root en el host si existe una fuga del namespace. Un proceso comprometido corriendo como `springboot` o `nginx` tiene acceso únicamente a los archivos bajo `/app` o los directorios de Nginx, sin capacidad de escalar privilegios en el sistema host.

### 3.3 Orquestación con Docker Compose

**Red bridge aislada:**

```yaml
networks:
  network_innovatech:
    driver: bridge
```

Todos los servicios pertenecen a `network_innovatech`. Docker crea una red virtual privada donde los contenedores se resuelven por nombre de servicio (e.g., `mysql-db`, `backend1`). Ningún servicio de base de datos está accesible desde fuera de esta red; el mapeo `3306:3306` puede eliminarse en producción para un aislamiento completo.

**Mapeo de puertos:**

| Servicio | Puerto externo | Puerto interno | Justificación |
|---|---|---|---|
| Frontend | 80 | 80 | Acceso HTTP público desde EC2/navegador |
| Backend Despachos | 8081 | 8081 | Corresponde a `server.port=8081` en properties |
| Backend Ventas | 8082 | 8080 | Evita colisión con el puerto por defecto de Spring |
| MySQL | 3306 | 3306 | Solo necesario en desarrollo local |

**Dependencias con `depends_on`:**

```yaml
backend1:
  depends_on:
    - mysql-db

frontend:
  depends_on:
    - backend1
    - backend2
```

`depends_on` garantiza el orden de arranque de contenedores. Para mayor robustez en producción se recomienda complementar con `healthcheck` en MySQL para asegurar que la base de datos esté lista para aceptar conexiones antes de que los backends intenten conectarse (Docker Compose `depends_on` por defecto solo espera que el contenedor esté *started*, no *healthy*).

---

## 4. Persistencia de Datos — Justificación IE2

El Compose define dos estrategias de persistencia diferenciadas según el tipo de dato:

```yaml
volumes:
  mysql_data:          # Named volume — datos de negocio
    driver: local

services:
  mysql-db:
    volumes:
      - mysql_data:/var/lib/mysql          # Named volume

  backend1:
    volumes:
      - ./back-Despachos_SpringBoot/logs:/app/logs   # Bind mount

  backend2:
    volumes:
      - ./back-Ventas_SpringBoot/logs:/app/logs      # Bind mount
```

**Named Volume (`mysql_data`) — Base de datos:**

El volumen `mysql_data` es gestionado por el daemon de Docker y almacenado en `/var/lib/docker/volumes/`. Su ciclo de vida es independiente del contenedor: un `docker-compose down` no lo elimina (requiere `--volumes` explícito). Esto asegura que los datos transaccionales sobrevivan reinicios, actualizaciones de imagen y recreaciones de contenedor. En AWS EC2, este volumen reside en el disco EBS adjunto a la instancia, por lo que persiste incluso ante reinicios del host. Para producción real se recomienda migrar a AWS RDS, que delega la persistencia, backups y alta disponibilidad al servicio administrado.

**Bind Mount (`./logs:/app/logs`) — Archivos de Log:**

Los logs se mapean a directorios del host mediante bind mount, lo que permite su acceso directo sin entrar al contenedor. La variable de entorno `LOGGING_FILE_NAME=/app/logs/backend.log` instruye a Spring Boot para escribir en esa ruta, que queda expuesta en el host bajo `./back-*/logs/`. Esto facilita la integración con agentes de monitoreo (CloudWatch Agent, Filebeat) que leen archivos del filesystem del host.

| Característica | Named Volume | Bind Mount |
|---|---|---|
| Gestión del ciclo de vida | Docker Daemon | Sistema operativo host |
| Portabilidad | Alta (independiente del path) | Baja (path absoluto del host) |
| Uso recomendado | Datos críticos (DB) | Logs, configuración de desarrollo |
| Supervive `docker-compose down` | Sí (sin `--volumes`) | N/A (es el filesystem del host) |

---

## 5. Pipeline CI/CD — Justificación IE3

El archivo `.github/workflows/deploy.yml` implementa un pipeline en dos jobs secuenciales activado exclusivamente en la rama `deploy`:

```yaml
on:
  push:
    branches:
      - deploy
```

La rama `deploy` actúa como gate de producción. El código en `main` o ramas de feature no dispara despliegues, aislando el entorno productivo de cambios no validados.

### Job 1: `build-and-publish`

```
Checkout → Login Docker Hub → Build imagen → Push Docker Hub
```

Se construyen y publican las tres imágenes de forma independiente:

```bash
docker build -t $DOCKERHUB_USERNAME/innovatech-frontend:latest ./frontend_devops
docker push $DOCKERHUB_USERNAME/innovatech-frontend:latest

docker build -t $DOCKERHUB_USERNAME/innovatech-backend1:latest ./back-Despachos_SpringBoot/...
docker push $DOCKERHUB_USERNAME/innovatech-backend1:latest

docker build -t $DOCKERHUB_USERNAME/innovatech-backend2:latest ./back-Ventas_SpringBoot/...
docker push $DOCKERHUB_USERNAME/innovatech-backend2:latest
```

Docker Hub actúa como registro de imágenes. El tag `:latest` permite que la EC2 siempre descargue la versión más reciente sin modificar el Compose.

### Job 2: `deploy-to-ec2`

```
Checkout → SCP docker-compose.yml → SSH → sed patch → docker-compose pull → docker-compose up -d
```

```yaml
needs: build-and-publish   # Solo ejecuta si Job 1 tuvo éxito
```

El deploy en EC2 se realiza en dos pasos:

1. **`scp-action`**: Copia únicamente el `docker-compose.yml` al servidor. La EC2 no requiere el código fuente; solo necesita las referencias a imágenes y la configuración de orquestación.

2. **`ssh-action`**: Se conecta y ejecuta el script de actualización:

```bash
# Reemplaza directivas 'build:' por 'image:' apuntando a Docker Hub
sed -i 's|build: ./frontend_devops|image: $DOCKERHUB_USERNAME/innovatech-frontend:latest|g' docker-compose.yml
# ... idem para backend1 y backend2

docker-compose down || true    # Detiene sin fallar si no hay stack activo
docker-compose pull            # Descarga las imágenes actualizadas
docker-compose up -d           # Levanta el stack en background
```

La transformación `sed` es necesaria porque el `docker-compose.yml` local usa `build:` (construye desde código fuente), mientras que en EC2 debe usar `image:` (descarga imagen pre-construida de Docker Hub). Esta estrategia evita mantener dos archivos Compose distintos.

**Gestión de secretos con GitHub Secrets:**

Ninguna credencial está hardcodeada en el repositorio. Todos los valores sensibles se inyectan en tiempo de ejecución desde el vault de GitHub:

| Secret | Propósito |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario Docker Hub para push/pull |
| `DOCKERHUB_TOKEN` | Token de acceso (no contraseña) |
| `EC2_HOST` | IP pública o DNS de la instancia |
| `SSH_PRIVATE_KEY` | Clave privada PEM para autenticación SSH |

Los Secrets de GitHub están cifrados en reposo y nunca aparecen en logs de workflow, incluso si se intenta hacer `echo` de ellos.

---

## 6. Infraestructura en AWS

**Topología de red:**

```
Internet
    │
    ▼
[Security Group: SG-Public]
    │  Inbound: TCP 80 (0.0.0.0/0)
    │  Inbound: TCP 22 (IP del equipo DevOps)
    ▼
[EC2 Instance — Amazon Linux 2]
    │  Docker Compose Stack
    │  Frontend :80 → público
    │  Backend  :8081, :8082 → interno (red bridge)
    │  MySQL    :3306 → interno (red bridge)
    ▼
[Security Group: SG-Private] (si se usa RDS)
    │  Inbound: TCP 3306 (solo desde SG-Public)
    ▼
[AWS RDS MySQL] (entorno producción real)
```

**Frontend — exposición pública:**

El contenedor `innovatech_frontend` mapea el puerto 80 del host, que el Security Group permite desde `0.0.0.0/0`. Nginx sirve la SPA estática y actúa como reverse proxy hacia los backends. Los usuarios finales acceden únicamente a este punto de entrada.

**Backends — acceso privado:**

Los puertos 8081 y 8082 **no están abiertos en el Security Group**. La comunicación entre el frontend Nginx y los backends ocurre dentro de la red `network_innovatech` (bridge de Docker), que es invisible desde Internet. El atacante que comprometiera el puerto 80 aún tendría que atravesar la red interna del stack para alcanzar los backends.

**Base de datos — completamente aislada:**

El puerto 3306 de MySQL no está expuesto en producción (eliminar el mapeo `3306:3306` del Compose). En el escenario con AWS RDS, el SG del RDS permite tráfico MySQL solo desde el SG de la EC2, siguiendo el principio de mínimo privilegio.

---

## 7. Guía de Uso

### Prerrequisitos

- Docker Desktop >= 24 o Docker Engine + Compose Plugin
- Git

### Levantar el stack localmente

```bash
# 1. Clonar el repositorio
git clone <url-del-repo>
cd <nombre-del-repo>

# 2. Construir y levantar todos los servicios
docker compose up --build -d

# 3. Verificar que todos los contenedores están corriendo
docker compose ps
```

### Acceso a servicios

| Servicio | URL |
|---|---|
| Frontend | http://localhost |
| Backend Despachos — Swagger UI | http://localhost:8081/swagger-ui.html |
| Backend Ventas — Swagger UI | http://localhost:8082/swagger-ui.html |

### Ver logs en tiempo real

```bash
# Logs de un servicio específico
docker compose logs -f backend1
docker compose logs -f backend2
docker compose logs -f frontend

# Logs de todos los servicios
docker compose logs -f
```

Los logs de los backends también quedan escritos en el filesystem del host:

```bash
tail -f ./back-Despachos_SpringBoot/logs/backend.log
tail -f ./back-Ventas_SpringBoot/logs/backend.log
```

### Detener el stack

```bash
# Detener sin eliminar datos
docker compose down

# Detener y eliminar volumen de base de datos (elimina todos los datos)
docker compose down --volumes
```

### Forzar un despliegue a producción

```bash
# Desde la rama de desarrollo, hacer merge a deploy
git checkout deploy
git merge main
git push origin deploy
# El pipeline CI/CD se activa automáticamente
```

---

## Estructura del Monorepo

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml               # Pipeline CI/CD
├── back-Despachos_SpringBoot/
│   └── Springboot-API-REST-DESPACHO/
│       ├── Dockerfile
│       ├── pom.xml
│       └── src/
├── back-Ventas_SpringBoot/
│   └── Springboot-API-REST/
│       ├── dockerfile
│       ├── pom.xml
│       └── src/
├── frontend_devops/
│   ├── dockerfile
│   ├── package.json
│   ├── pnpm-lock.yaml
│   └── front_despacho-main/
│       └── src/
└── docker-compose.yml
```
