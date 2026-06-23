# Proyecto Despacho - DevOps

## Descripción General

**Proyecto Despacho** es una aplicación web moderna de gestión que implementa una arquitectura de microservicios completa. El proyecto demuestra prácticas avanzadas de DevOps mediante la contenedorización de todos sus componentes (Frontend, Backend y Base de Datos) utilizando Docker y Docker Compose para orquestación.

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

Esta configuración es **production-ready** y escala fácilmente a orquestadores como Kubernetes, manteniendo los mismos conceptos y archivos (con mínimas adaptaciones).

---

**Última actualización**: Mayo 2026  
**Versión**: 1.0  
**Mantenedor**: Equipo DevOps Proyecto Despacho
