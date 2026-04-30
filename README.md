# Arquitectura y Despliegue en Docker - Sistema de Inventarios Petyzoos

Este documento sirve como guía definitiva para entender cómo funciona la arquitectura de contenedores de tu proyecto, cómo se conectan los distintos componentes y qué hace cada línea de configuración.

## 1. Arquitectura General del Sistema

Tu proyecto utiliza una arquitectura de microservicios contenerizados, orquestados mediante **Docker Compose**. Esto significa que en lugar de instalar Angular, Java y MySQL directamente en tu computadora, cada componente vive dentro de su propia "burbuja" o contenedor.

A continuación, un diagrama de cómo se comunican estos contenedores:

```mermaid
graph TD
    Client((Navegador Web / Usuario))
    
    subgraph "Docker Engine (Servidor/Computadora)"
        subgraph "Red Interna: inventario-net"
            Frontend[Frontend<br/>Angular + Nginx<br/>Puerto Expuesto: 3000]
            Backend[Backend<br/>Spring Boot + JRE 21<br/>Puerto Expuesto: 5000]
            DB[(Base de Datos<br/>MySQL 8.0<br/>Puerto Expuesto: 3306)]
        end
        
        Volumen[(Volumen Persistente<br/>petyzoos-db-data)]
    end
    
    Client -->|Carga la Interfaz Gráfica<br/>http://localhost:3000| Frontend
    Client -->|Envía y Recibe Datos (API)<br/>http://localhost:5000| Backend
    Backend -->|Consultas SQL<br/>jdbc:mysql://db:3306| DB
    DB <-->|Guarda la información físicamente| Volumen
```

### ¿Cómo funciona la comunicación?
- **Red Aislada (`inventario-net`)**: Docker crea una red privada virtual exclusiva para tu proyecto. Esto genera un "DNS interno". Gracias a esto, tu Backend puede conectarse a la base de datos simplemente usando la palabra `db` en lugar de una dirección IP numérica.
- **Volúmenes (`db-data`)**: Los contenedores son efímeros (si se borran, se pierde todo su contenido). Para evitar que los datos de MySQL desaparezcan al apagar el contenedor, se usa un **Volumen**. Este disco virtual guarda los datos de forma permanente en tu computadora física.

---

## 2. Levantando el Proyecto (`docker-compose.yml`)

El archivo `docker-compose.yml` es el **director de orquesta**. Su trabajo es leer la configuración y levantar todos los contenedores en el orden correcto.

> [!TIP]
> **Comando Principal**
> Para levantar todo el proyecto, debes abrir una terminal en la carpeta principal y ejecutar:
> ```bash
> docker compose up -d --build
> ```
> - **`up`**: Crea e inicia los contenedores.
> - **`-d`**: *Detached mode*, deja los contenedores corriendo en segundo plano.
> - **`--build`**: Obliga a reconstruir las imágenes si hubo cambios en el código.

### ¿En qué orden se levantan?
Gracias a la propiedad `depends_on`, Docker Compose sabe exactamente qué levantar primero:
1. Nace la **Base de datos (`db`)**.
2. El **Backend** espera hasta que la Base de datos esté sana y responda a las pruebas de conexión (*healthcheck*). Luego, arranca el Backend.
3. El **Frontend** espera a que el Backend haya arrancado para encenderse.

---

## 3. Explicación línea por línea de los Archivos Docker

A continuación, analizamos a fondo qué hace cada archivo que construye tu proyecto.

### A. Frontend (Angular) - `frontend-inventario/Dockerfile`
Utiliza **Multi-stage build**. Esto significa que primero usa una computadora con Node.js para compilar (ensamblar) el código, y luego bota esa computadora y solo se queda con el HTML/CSS resultante metido en un servidor web ultraligero (Nginx).

```dockerfile
# ── ETAPA 1: BUILD ──
# 1. Traemos una imagen base con Node.js (versión ligera Alpine). La llamamos "build".
FROM node:22-alpine AS build

# 2. Entramos a una carpeta "/app" dentro de la computadora virtual.
WORKDIR /app

# 3. Copiamos SOLO la lista de librerías. (Truco para que Docker guarde esto en memoria caché).
COPY package.json package-lock.json ./

# 4. Instalamos las dependencias de forma limpia y rápida.
RUN npm ci --no-audit --no-fund

# 5. Copiamos todo el resto del código fuente del frontend.
COPY . .

# 6. Compilamos Angular para producción. (Se generan los archivos listos para internet).
RUN npx ng build --configuration production

# ── ETAPA 2: PRODUCCIÓN ──
# 7. Desechamos Node.js y traemos Nginx (servidor web súper veloz y ligero).
FROM nginx:alpine AS production

# 8. Copiamos nuestra configuración de rutas personalizada para el servidor.
COPY nginx.conf /etc/nginx/conf.d/default.conf

# 9. ¡EL PASO CLAVE! Rescatamos los archivos HTML/JS generados en la "ETAPA 1" y los ponemos en la carpeta pública de Nginx.
COPY --from=build /app/dist/frontend-inventario/browser /usr/share/nginx/html

# 10. Avisamos que este contenedor recibirá visitas por el puerto 80.
EXPOSE 80

# 11. Encendemos el servidor web para que nunca se apague.
CMD ["nginx", "-g", "daemon off;"]
```

### B. Backend (Spring Boot) - `backend/Dockerfile`
Al igual que el Frontend, utiliza múltiples etapas para no enviar al servidor final herramientas de compilación pesadas, reduciendo el tamaño de la imagen.

```dockerfile
# ── ETAPA 1: BUILD ──
# 1. Traemos una imagen con Maven y Java 21 para poder compilar.
FROM maven:3.9-eclipse-temurin-21-alpine AS build

# 2. Entramos a la carpeta de trabajo.
WORKDIR /app

# 3. Copiamos la configuración de librerías.
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .

# 4. Descargamos las librerías de internet.
RUN mvn dependency:go-offline -B

# 5. Copiamos nuestro código fuente Java.
COPY src ./src

# 6. Compilamos todo nuestro código en un archivo ".jar". Nos saltamos los tests por rapidez.
RUN mvn clean package -DskipTests --no-transfer-progress

# ── ETAPA 2: PRODUCCIÓN ──
# 7. Desechamos Maven y traemos un "JRE" (Solo lo estrictamente necesario para correr Java).
FROM eclipse-temurin:21-jre-alpine AS production

# 8. Entramos a la carpeta final de trabajo.
WORKDIR /app

# 9. Por medidas de ciberseguridad, creamos un usuario común (no administrador).
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 10. Rescatamos el archivo ".jar" de la ETAPA 1.
COPY --from=build /app/target/*.jar app.jar

# 11. Le damos permiso al nuevo usuario para manejar este archivo.
RUN chown appuser:appgroup app.jar

# 12. A partir de aquí, ejecutamos todo con ese usuario seguro.
USER appuser

# 13. Avisamos que el backend atiende por el puerto 8080.
EXPOSE 8080

# 14. HEALTHCHECK: Cada 30s Docker se asegura que el backend siga vivo haciéndole ping a la ruta /api/dashboard.
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/api/dashboard || exit 1

# 15. Encendemos el servidor y le avisamos que use la configuración especial de Docker.
CMD ["java", "-jar", "app.jar", "--spring.profiles.active=docker"]
```

### C. Base de Datos (MySQL) - Directamente en `docker-compose.yml`
La base de datos **no necesita un Dockerfile** porque no compilamos código propio; solo configuramos un programa ya existente. Todo se define en el archivo `docker-compose.yml`:

```yaml
  db:
    # 1. Descarga el programa MySQL versión 8 directo de internet.
    image: mysql:8.0                     
    
    # 2. Le inyecta contraseñas y configuraciones iniciales.
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASS}
      MYSQL_DATABASE: ${DB_NAME}
      # ...
      
    # 3. Expone el puerto por el que se conectan los clientes SQL (ej. DBeaver).
    ports:
      - "${DB_PORT:-3306}:3306"          
      
    # 4. Configuración de discos virtuales.
    volumes:
      # Guarda los datos físicamente para no perderlos si el contenedor se apaga.
      - db-data:/var/lib/mysql           
      
      # Inyecta tu script "init.sql". MySQL lo detecta y lo ejecuta automáticamente la 1ra vez, creando tus tablas sin que tú muevas un dedo.
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql  
      
    # 5. Verifica que la BD esté encendida antes de que el Backend intente conectarse.
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_ROOT_PASS}"]
```

> [!NOTE]
> **Resumen del Flujo de Trabajo**
> Todo el proceso automatizado te evita tener que instalar software manualmente. Solo con el comando de Docker Compose, el sistema descarga dependencias, compila el Frontend, empaqueta el Backend, crea la Base de datos con sus tablas listas y conecta a los tres servicios de forma segura.
