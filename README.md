# Proyecto Despacho - DevOps

## Descripción General

**Proyecto Despacho** es una aplicación web moderna de gestión que implementa una arquitectura de microservicios completa. El proyecto demuestra prácticas avanzadas de DevOps mediante la contenedorización, orquestación y automatización de despliegues en infraestructura cloud.

La solución permite desplegar y gestionar de forma eficiente un sistema integrado de tres capas, asegurando portabilidad, reproducibilidad y escalabilidad en diferentes entornos.

---

## Arquitectura del Proyecto

La arquitectura sigue el patrón de **tres capas** con contenedorización completa:

| Servicio | Tecnología | Puerto | Función |
|----------|-----------|--------|---------|
| **Frontend** | React + Vite | 5173 | Interfaz de usuario moderna y responsiva |
| **Backend** | Spring Boot + Maven | 8081 | API REST para lógica de negocio |
| **Base de Datos** | MySQL 8.0 | 3306 | Persistencia de datos relacional |

### Componentes

- **Frontend (React/Vite)**: Aplicación cliente con interfaz moderna, construida con React y Vite para rendimiento optimizado.
- **Backend (Spring Boot)**: Servidor API REST que expone endpoints para gestión de datos, implementado con Spring Boot y Maven.
- **Database (MySQL 8.0)**: Sistema gestor de base de datos relacional ejecutado en contenedor Docker con volúmenes persistentes.

---

## Estructura de Carpetas

```
proyecto-despacho-devops/
├── front_despacho/                          # Frontend React/Vite
│   ├── Dockerfile
│   ├── package.json
│   ├── vite.config.js
│   └── src/
│
├── back-Despachos_SpringBoot/               # Backend Spring Boot
│   └── Springboot-API-REST-DESPACHO/
│       ├── Dockerfile
│       ├── pom.xml
│       ├── src/
│       └── target/
│
├── .github/
│   └── workflows/                           # Workflows de GitHub Actions
│       ├── build-and-push-backend.yml       # CI/CD Backend ECS/ECR
│       └── build-and-push-frontend.yml      # CI/CD Frontend ECS/ECR
│
├── docker-compose.yml                       # Orquestación de servicios
├── .gitignore
└── README.md                                 # Este archivo
```

---

## Dockerización

### Frontend Dockerfile

**Ubicación**: `front_despacho/Dockerfile`

El Dockerfile del frontend utiliza un enfoque de **multi-stage build**:

```dockerfile
# Stage 1: Construcción con Node
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Servicio con Nginx
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 5173
CMD ["nginx", "-g", "daemon off;"]
```

**Características**:
- **Etapa 1 (Construcción)**: Utiliza imagen de Node.js Alpine para instalar dependencias y compilar la aplicación React/Vite.
- **Etapa 2 (Servicio)**: Utiliza Nginx Alpine para servir los archivos estáticos compilados de forma eficiente.
- **Ventajas**: Imagen final ligera (~20MB), optimizada solo con lo necesario para producción.

---

### Backend Dockerfile

**Ubicación**: `back-Despachos_SpringBoot/Springboot-API-REST-DESPACHO/Dockerfile`

El Dockerfile del backend implementa **multi-stage build** con separación de roles:

```dockerfile
# Stage 1: Compilación con Maven y Temurin
FROM maven:3.9-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Ejecución con JRE
FROM temurin:21-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
USER appuser
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Características**:
- **Etapa 1 (Compilación)**: Maven descarga dependencias y compila la aplicación Spring Boot en un JAR ejecutable.
- **Etapa 2 (Ejecución)**: Utiliza JRE Temurin (JDK mínimo para ejecución) reduciendo el tamaño de imagen.
- **Seguridad**: Crea usuario no-root (`appuser`) para ejecutar la aplicación, mejorando la postura de seguridad.
- **Multi-stage**: La imagen final contiene solo el JAR compilado, eliminando herramientas de build innecesarias (~80MB vs 500MB+).

---

## Docker Compose

**Ubicación**: `back-Despachos_SpringBoot/Springboot-API-REST-DESPACHO/docker-compose.yml`

Archivo de orquestación que define y conecta los tres servicios:

```yaml
version: '3.8'

services:
  # Servicio de Base de Datos MySQL
  mysql-despacho:
    image: mysql:8.0
    container_name: mysql-despacho
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - despacho-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Servicio Backend Spring Boot
  despacho-backend:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: despacho-backend-container
    depends_on:
      mysql-despacho:
        condition: service_healthy
    environment:
      DB_ENDPOINT: mysql-despacho
      DB_PORT: 3306
      DB_NAME: ${DB_NAME}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
    ports:
      - "8081:8081"
    networks:
      - despacho-network

  # Servicio Frontend React/Vite
  despacho-frontend:
    build:
      context: ../../front_despacho
      dockerfile: Dockerfile
    container_name: despacho-frontend-container
    depends_on:
      - despacho-backend
    ports:
      - "5173:5173"
    environment:
      VITE_API_URL: http://despacho-backend:8081
    networks:
      - despacho-network

volumes:
  mysql_data:
    driver: local

networks:
  despacho-network:
    driver: bridge
```

### Explicación de Conceptos DevOps

**Volumen `mysql_data`**: Almacenamiento persistente que mantiene los datos de MySQL incluso cuando el contenedor se detiene o elimina, garantizando la persistencia de datos entre reinicios.

**Red `despacho-network`**: Bridge network que permite comunicación segura y aislada entre contenedores usando DNS interno (ej: `mysql-despacho` desde el backend).

**Healthcheck**: Verifica que MySQL esté listo antes de iniciar el backend, asegurando dependencias correctas.

---

## CI/CD con GitHub Actions - ECS y ECR

### Descripción General

El proyecto implementa una **tubería de integración continua y despliegue continuo (CI/CD)** mediante **GitHub Actions** que automatiza:

1. **Build de imágenes Docker** del Frontend y Backend
2. **Push a Amazon ECR** (Elastic Container Registry)
3. **Despliegue automático en ECS** (Elastic Container Service) en AWS

### Arquitectura CI/CD

```
Push a rama (main/develop)
    ↓
GitHub Actions se dispara
    ↓
Build imagen Docker
    ↓
Push a Amazon ECR
    ↓
Actualizar servicio en ECS
    ↓
Despliegue automático
```

---

### Workflow - Backend (ECS/ECR)

**Ubicación**: `.github/workflows/build-and-push-backend.yml`

Este workflow automatiza el despliegue del Backend en AWS ECS:

```yaml
name: Build and Push Backend to ECR

on:
  push:
    branches:
      - main
      - develop
    paths:
      - 'back-Despachos_SpringBoot/**'
      - '.github/workflows/build-and-push-backend.yml'

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout código
        uses: actions/checkout@v3

      - name: Configurar AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login a Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build imagen Docker del Backend
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: despacho-backend
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            -f back-Despachos_SpringBoot/Springboot-API-REST-DESPACHO/Dockerfile \
            back-Despachos_SpringBoot/Springboot-API-REST-DESPACHO/

      - name: Push imagen a Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: despacho-backend
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: Actualizar servicio ECS
        env:
          ECS_CLUSTER: despacho-cluster
          ECS_SERVICE: despacho-backend-service
          ECS_TASK_DEFINITION: despacho-backend-task
          AWS_REGION: us-east-1
        run: |
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service $ECS_SERVICE \
            --force-new-deployment \
            --region $AWS_REGION
```

**Características principales**:
- **Trigger**: Se dispara automáticamente en push a `main` o `develop` cuando hay cambios en el directorio del Backend
- **Build**: Construye imagen Docker usando el Dockerfile del Backend
- **Push**: Envía imagen a Amazon ECR con tags `latest` y `${{ github.sha }}`
- **Despliegue**: Actualiza automáticamente el servicio ECS para usar la nueva imagen

---

### Workflow - Frontend (ECS/ECR)

**Ubicación**: `.github/workflows/build-and-push-frontend.yml`

Este workflow automatiza el despliegue del Frontend en AWS ECS:

```yaml
name: Build and Push Frontend to ECR

on:
  push:
    branches:
      - main
      - develop
    paths:
      - 'front_despacho/**'
      - '.github/workflows/build-and-push-frontend.yml'

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout código
        uses: actions/checkout@v3

      - name: Configurar AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login a Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build imagen Docker del Frontend
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: despacho-frontend
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            -f front_despacho/Dockerfile \
            front_despacho/

      - name: Push imagen a Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: despacho-frontend
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: Actualizar servicio ECS
        env:
          ECS_CLUSTER: despacho-cluster
          ECS_SERVICE: despacho-frontend-service
          ECS_TASK_DEFINITION: despacho-frontend-task
          AWS_REGION: us-east-1
        run: |
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service $ECS_SERVICE \
            --force-new-deployment \
            --region $AWS_REGION
```

**Características principales**:
- **Trigger**: Se dispara automáticamente en push a `main` o `develop` cuando hay cambios en el directorio del Frontend
- **Build**: Construye imagen Docker usando el Dockerfile del Frontend (multi-stage con Nginx)
- **Push**: Envía imagen a Amazon ECR con tags `latest` y `${{ github.sha }}`
- **Despliegue**: Actualiza automáticamente el servicio ECS para usar la nueva imagen

---

### Configuración de Secretos en GitHub

Para que los workflows funcionen correctamente, debes configurar los siguientes secretos en GitHub:

**Ubicación**: Settings → Secrets and variables → Actions → New repository secret

| Secreto | Descripción | Ejemplo |
|---------|-------------|---------|
| `AWS_ACCESS_KEY_ID` | ID de clave de acceso AWS | `AKIAIOSFODNN7EXAMPLE` |
| `AWS_SECRET_ACCESS_KEY` | Clave de acceso secreta AWS | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |

**Pasos para configurar**:

1. Ve a tu repositorio en GitHub
2. Haz clic en **Settings**
3. Selecciona **Secrets and variables** → **Actions**
4. Haz clic en **New repository secret**
5. Agrega cada secreto con su valor correspondiente

---

### Requisitos Previos en AWS

Antes de ejecutar los workflows, debes tener configurado en AWS:

#### 1. **Amazon ECR - Repositorios**

Crea dos repositorios en ECR:

```bash
aws ecr create-repository --repository-name despacho-backend --region us-east-1
aws ecr create-repository --repository-name despacho-frontend --region us-east-1
```

#### 2. **Amazon ECS - Cluster**

Crea un cluster ECS:

```bash
aws ecs create-cluster --cluster-name despacho-cluster --region us-east-1
```

#### 3. **ECS Task Definition - Backend**

```json
{
  "family": "despacho-backend-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "despacho-backend",
      "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/despacho-backend:latest",
      "portMappings": [
        {
          "containerPort": 8081,
          "hostPort": 8081,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "DB_ENDPOINT",
          "value": "mysql-despacho"
        },
        {
          "name": "DB_PORT",
          "value": "3306"
        },
        {
          "name": "DB_NAME",
          "value": "despacho_db"
        }
      ]
    }
  ]
}
```

#### 4. **ECS Task Definition - Frontend**

```json
{
  "family": "despacho-frontend-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "despacho-frontend",
      "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/despacho-frontend:latest",
      "portMappings": [
        {
          "containerPort": 5173,
          "hostPort": 5173,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "VITE_API_URL",
          "value": "http://despacho-backend:8081"
        }
      ]
    }
  ]
}
```

#### 5. **ECS Service - Backend**

```bash
aws ecs create-service \
  --cluster despacho-cluster \
  --service-name despacho-backend-service \
  --task-definition despacho-backend-task \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxxxxxx],securityGroups=[sg-xxxxxxxxx],assignPublicIp=ENABLED}" \
  --region us-east-1
```

#### 6. **ECS Service - Frontend**

```bash
aws ecs create-service \
  --cluster despacho-cluster \
  --service-name despacho-frontend-service \
  --task-definition despacho-frontend-task \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxxxxxx],securityGroups=[sg-xxxxxxxxx],assignPublicIp=ENABLED}" \
  --region us-east-1
```

---

### Flujo de Despliegue Automatizado

1. **Desarrollador hace push** a rama `main` o `develop`
2. **GitHub Actions dispara** el workflow correspondiente
3. **Workflow ejecuta**:
   - Checkout del código
   - Autenticación en AWS ECR
   - Build de la imagen Docker
   - Push a ECR con tags `latest` y SHA del commit
4. **ECS actualiza automáticamente** el servicio con la nueva imagen
5. **Despliegue en vivo** en segundos/minutos

---

### Monitoreo de Workflows

Puedes ver el estado de los workflows en:

**GitHub**: Actions → Selecciona el workflow → Ver detalles de la ejecución

**Indicadores de estado**:
- 🟢 **Verde**: Workflow completado exitosamente
- 🟡 **Amarillo**: Workflow en ejecución
- 🔴 **Rojo**: Workflow falló

---

## Variables de Entorno del Backend

El backend requiere las siguientes variables de entorno para conectarse a la base de datos:

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `DB_ENDPOINT` | Host del servidor MySQL | `mysql-despacho` |
| `DB_PORT` | Puerto de MySQL | `3306` |
| `DB_NAME` | Nombre de la base de datos | `despacho_db` |
| `DB_USERNAME` | Usuario de acceso a MySQL | `despacho_user` |
| `DB_PASSWORD` | Contraseña del usuario | `contraseña_segura` |

**Archivo `.env` (en la raíz del docker-compose)**:

```env
DB_ENDPOINT=mysql-despacho
DB_PORT=3306
DB_NAME=despacho_db
DB_USERNAME=despacho_user
DB_PASSWORD=contraseña_segura_aqui
```

---

## Cómo Levantar el Proyecto Localmente

### Prerrequisitos

- **Docker Desktop** instalado y ejecutándose
- Git clonado: `proyecto-despacho-devops`
- Variables de entorno configuradas

### Pasos de Instalación

#### 1. Preparar el Entorno

```bash
# Navegar a la carpeta del backend (donde está docker-compose.yml)
cd back-Despachos_SpringBoot/Springboot-API-REST-DESPACHO

# Crear archivo .env con las variables de entorno
cat > .env << EOF
DB_ENDPOINT=mysql-despacho
DB_PORT=3306
DB_NAME=despacho_db
DB_USERNAME=despacho_user
DB_PASSWORD=tu_contraseña_segura
EOF
```

#### 2. Iniciar la Base de Datos

```bash
# Levantar MySQL con Docker Compose (modo background)
docker-compose up -d mysql-despacho

# Verificar que MySQL está corriendo
docker ps | grep mysql-despacho
```

#### 3. Compilar e Iniciar Backend

```bash
# Construir imagen del backend
docker-compose build despacho-backend

# Ejecutar contenedor del backend
docker-compose up -d despacho-backend

# Verificar logs del backend
docker logs despacho-backend-container
```

#### 4. Compilar e Iniciar Frontend

```bash
# Navegar a la carpeta del frontend
cd ../../../front_despacho

# Construir imagen del frontend
docker build -t despacho-frontend .

# Ejecutar contenedor del frontend
docker run -d \
  --name despacho-frontend-container \
  -p 5173:5173 \
  --network despacho-network \
  -e VITE_API_URL=http://despacho-backend:8081 \
  despacho-frontend
```

#### 5. Verificar que Todo Está Corriendo

```bash
# Listar todos los contenedores en ejecución
docker ps

# Resultado esperado:
# CONTAINER ID   IMAGE                            STATUS          PORTS
# xxx            mysql:8.0                        Up 2 minutes    3306/tcp
# yyy            despacho-backend                 Up 1 minute     0.0.0.0:8081->8081/tcp
# zzz            despacho-frontend                Up 30 seconds   0.0.0.0:5173->5173/tcp
```

---

## URLs de Acceso

Una vez que todos los servicios estén corriendo, accede a:

| Servicio | URL |
|----------|-----|
| **Frontend** | [http://localhost:5173](http://localhost:5173) |
| **Backend - API REST** | [http://localhost:8081/api](http://localhost:8081/api) |
| **Backend - Swagger UI** | [http://localhost:8081/swagger-ui/index.html](http://localhost:8081/swagger-ui/index.html) |
| **Base de Datos (Host Local)** | `localhost:3306` |

---

## Evidencias de Funcionamiento

Para validar que el proyecto está funcionando correctamente, adjunta las siguientes capturas:

1. **Docker Desktop - Contenedores Activos**
   - Captura de pantalla de Docker Desktop mostrando los tres contenedores en estado "Running"
   - Incluir: `mysql-despacho`, `despacho-backend-container`, `despacho-frontend-container`

2. **Terminal - Docker ps**
   ```bash
   docker ps
   ```
   - Captura mostrando los tres contenedores con sus puertos mapeados

3. **Frontend**
   - Captura de navegador en `http://localhost:5173`
   - Mostrar interfaz cargada correctamente

4. **Backend - Swagger UI**
   - Captura de navegador en `http://localhost:8081/swagger-ui/index.html`
   - Mostrar endpoints disponibles de la API

5. **Conexión de Base de Datos**
   - Captura de herramienta como MySQL Workbench o DBeaver conectada a `localhost:3306`
   - Mostrar base de datos y tablas creadas

---

## Relación con Prácticas DevOps

Este proyecto implementa principios clave de DevOps:

### 1. **Contenedorización**
- Cada componente (Frontend, Backend, BD) está contenedorizado independientemente
- Garantiza consistencia entre entornos de desarrollo, testing y producción
- Facilita el despliegue rápido y reproducible

### 2. **Persistencia de Datos**
- Volumen `mysql_data` asegura que los datos persisten más allá del ciclo de vida del contenedor
- Implementa backup automático mediante volúmenes de Docker
- Cumple con SLA de disponibilidad de datos

### 3. **Gestión de Variables de Entorno**
- Configuración externa mediante `.env` permite diferentes valores por entorno
- Separación clara entre código y configuración
- Seguridad: credenciales no hardcodeadas en código fuente

### 4. **Separación de Servicios**
- Arquitectura de microservicios permite escalar componentes independientemente
- Cada servicio tiene responsabilidad única (Single Responsibility Principle)
- Red aislada `despacho-network` para comunicación segura

### 5. **Portabilidad y Reproducibilidad**
- `docker-compose.yml` define el ambiente completo
- Un comando (`docker-compose up`) despliega todo el stack
- Funciona idéntico en laptop, servidor CI/CD o cloud

### 6. **Health Checks y Dependencias**
- Healthcheck en MySQL verifica disponibilidad antes de iniciar Backend
- `depends_on` asegura orden correcto de inicio
- Mejora confiabilidad del deployment

### 7. **Seguridad**
- Backend ejecuta con usuario no-root en contenedor
- Redes aisladas limitan exposición
- Multigrana build minimiza superficie de ataque

### 8. **Integración Continua y Despliegue Continuo (CI/CD)**
- Workflows automáticos en GitHub Actions para despliegues
- Build y push automático de imágenes a ECR
- Despliegue sin intervención manual en ECS
- Garantiza consistencia y rapidez en los despliegues a producción

---

## Comandos Útiles

### Gestión de Contenedores

```bash
# Listar contenedores corriendo
docker ps

# Listar todos los contenedores (incluyendo detenidos)
docker ps -a

# Ver logs en tiempo real
docker logs -f despacho-backend-container

# Ver últimas 100 líneas de logs
docker logs --tail 100 despacho-backend-container

# Detener un contenedor
docker stop despacho-backend-container

# Iniciar un contenedor detenido
docker start despacho-backend-container

# Reiniciar un contenedor
docker restart despacho-backend-container

# Eliminar un contenedor
docker rm despacho-backend-container
```

### Gestión de Docker Compose

```bash
# Levantar todos los servicios
docker-compose up -d

# Detener todos los servicios
docker-compose down

# Detener servicios sin eliminar volúmenes
docker-compose stop

# Iniciar servicios detenidos
docker-compose start

# Reconstruir imágenes
docker-compose build

# Ver logs de todos los servicios
docker-compose logs -f

# Ejecutar comando en contenedor corriendo
docker-compose exec despacho-backend sh
```

### Gestión de Volúmenes

```bash
# Listar volúmenes
docker volume ls

# Inspeccionar volumen
docker volume inspect mysql_data

# Eliminar volumen (⚠️ elimina datos)
docker volume rm mysql_data
```

### Inspección de Red

```bash
# Listar redes
docker network ls

# Inspeccionar red
docker network inspect despacho-network
```

### Comandos AWS CLI para CI/CD

```bash
# Ver repositorios ECR
aws ecr describe-repositories --region us-east-1

# Ver imágenes en repositorio ECR
aws ecr describe-images --repository-name despacho-backend --region us-east-1

# Ver servicios ECS
aws ecs list-services --cluster despacho-cluster --region us-east-1

# Ver tareas en ejecución
aws ecs list-tasks --cluster despacho-cluster --region us-east-1

# Describir servicio ECS
aws ecs describe-services --cluster despacho-cluster --services despacho-backend-service --region us-east-1

# Ver logs de despliegue ECS
aws ecs describe-tasks --cluster despacho-cluster --tasks <TASK_ARN> --region us-east-1
```

---

## Conclusión Técnica

El **Proyecto Despacho** demuestra una implementación profesional de DevOps mediante:

- ✅ **Contenedorización completa** de los tres niveles de la arquitectura
- ✅ **Orquestación declarativa** con Docker Compose
- ✅ **Persistencia garantizada** de datos mediante volúmenes
- ✅ **Aislamiento de servicios** con redes personalizadas
- ✅ **Configuración externalizad**a en variables de entorno
- ✅ **Multi-stage builds** para imágenes optimizadas
- ✅ **Seguridad mejorada** con usuarios no-root
- ✅ **Health checks** para confiabilidad
- ✅ **Automatización CI/CD** con GitHub Actions, ECR y ECS
- ✅ **Despliegues sin intervención manual** en infraestructura cloud

Esta configuración es **production-ready** y escala fácilmente a orquestadores como Kubernetes, manteniendo los mismos conceptos y archivos (con mínimas adaptaciones).

---

**Última actualización**: Junio 2026  
**Versión**: 2.0  
**Mantenedor**: Equipo DevOps Proyecto Despacho
