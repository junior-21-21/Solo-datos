Arquitectura y Despliegue en Docker - Sistema de Inventarios Petyzoos   Este documento sirve como guía definitiva para entender cómo funciona la arquitectura de contenedores de tu proyecto, cómo se conectan los distintos componentes y qué hace cada línea de configuración.  1. Arquitectura General del Sistema   Tu proyecto utiliza una arquitectura de microservicios contenerizados, orquestados mediante Docker Compose. Esto significa que en lugar de instalar Angular, Java y MySQL directamente en tu computadora, cada componente vive dentro de su propia "burbuja" o contenedor.  A continuación, un diagrama de cómo se comunican estos contenedores:  Fragmento de códigograph TD
    Client((Navegador Web / Usuario))
    
    subgraph "Docker Engine (Servidor/Computadora)"
        subgraph "Red Interna: inventario-net"
            Frontend["Frontend<br/>Angular + Nginx<br/>Puerto Expuesto: 3000"]
            Backend["Backend<br/>Spring Boot + JRE 21<br/>Puerto Expuesto: 5000"]
            DB[("Base de Datos<br/>MySQL 8.0<br/>Puerto Expuesto: 3306")]
        end
        
        Volumen[("Volumen Persistente<br/>petyzoos-db-data")]
    end
    
    Client -->|"Carga la Interfaz Gráfica<br/>http://localhost:3000"| Frontend
    Client -->|"Envía y Recibe Datos (API)<br/>http://localhost:5000"| Backend
    Backend -->|"Consultas SQL<br/>jdbc:mysql://db:3306"| DB
    DB <-->|"Guarda la información físicamente"| Volumen
¿Cómo funciona la comunicación?   Red Aislada (inventario-net): Docker crea una red privada virtual exclusiva para tu proyecto. Esto genera un "DNS interno" que permite al Backend conectarse a la base de datos usando el nombre db en lugar de una IP.  Volúmenes (db-data): Se usa un Volumen nombrado (petyzoos-db-data) para que los datos de MySQL persistan permanentemente en el disco físico, evitando que se pierdan al apagar el contenedor.  2. Levantando el Proyecto (docker-compose.yml)   El archivo docker-compose.yml es el director de orquesta. Su trabajo es leer la configuración y levantar todos los contenedores en el orden correcto.  [!TIP]Comando PrincipalPara levantar todo el proyecto, debes abrir una terminal en la carpeta principal y ejecutar:Bashdocker compose up -d --build
up: Crea e inicia los contenedores.
- -d: Detached mode, corre en segundo plano.  --build: Obliga a reconstruir las imágenes si hubo cambios en el código.¿En qué orden se levantan?   Nace la Base de datos (db).El Backend espera hasta que la Base de datos esté sana y responda al healthcheck.  El Frontend espera a que el Backend haya arrancado para encenderse.  3. Explicación línea por línea de los Archivos Docker   A. Frontend (Angular) - frontend-inventario/Dockerfile   Utiliza Multi-stage build para separar el entorno pesado de compilación del servidor ligero de producción.  Dockerfile# ── ETAPA 1: BUILD ──
# 1. Imagen base con Node.js ligera (Alpine)[cite: 50].
FROM node:22-alpine AS build

# 2. Carpeta de trabajo interna[cite: 51].
WORKDIR /app

# 3. Copia de archivos de dependencias para aprovechar el caché de Docker[cite: 53, 68].
COPY package.json package-lock.json ./

# 4. Instalación limpia de dependencias[cite: 54].
RUN npm ci --no-audit --no-fund

# 5. Copia del código fuente[cite: 56].
COPY . .

# 6. Compilación de Angular para producción[cite: 57].
RUN npx ng build --configuration production

# ── ETAPA 2: PRODUCCIÓN ──
# 7. Servidor web Nginx ultraligero[cite: 59].
FROM nginx:alpine AS production

# 8. Configuración de rutas personalizada[cite: 60].
COPY nginx.conf /etc/nginx/conf.d/default.conf

# 9. Transferencia de archivos compilados a la carpeta pública de Nginx[cite: 61].
COPY --from=build /app/dist/frontend-inventario/browser /usr/share/nginx/html

# 10. Exposición del puerto 80[cite: 61].
EXPOSE 80

# 11. Ejecución de Nginx[cite: 62].
CMD ["nginx", "-g", "daemon off;"]
B. Backend (Spring Boot) - backend/Dockerfile   Optimiza el tamaño final usando un JRE (entorno de ejecución) en lugar del JDK completo.  Dockerfile# ── ETAPA 1: BUILD ──
# 1. Imagen con Maven y Java 21 para compilar[cite: 142].
FROM maven:3.9-eclipse-temurin-21-alpine AS build
WORKDIR /app

# 2. Preparación de herramientas de construcción[cite: 144, 145, 146].
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .

# 3. Descarga de dependencias en capa independiente[cite: 147, 169].
RUN mvn dependency:go-offline -B

# 4. Compilación del archivo .jar ejecutable[cite: 148, 149].
COPY src ./src
RUN mvn clean package -DskipTests --no-transfer-progress

# ── ETAPA 2: PRODUCCIÓN ──
# 5. Entorno de ejecución Java ligero[cite: 151].
FROM eclipse-temurin:21-jre-alpine AS production
WORKDIR /app

# 6. Seguridad: Creación de usuario sin privilegios de administrador[cite: 153, 255].
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 7. Rescate del .jar y asignación de permisos[cite: 154, 155].
COPY --from=build /app/target/*.jar app.jar
RUN chown appuser:appgroup app.jar

# 8. Ejecución con usuario seguro[cite: 156].
USER appuser
EXPOSE 8080

# 9. HEALTHCHECK: Monitoreo constante de la salud del servicio[cite: 158, 160].
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/api/dashboard || exit 1

# 10. Inicio de la aplicación con perfil Docker[cite: 161].
CMD ["java", "-jar", "app.jar", "--spring.profiles.active=docker"]
C. Base de Datos (MySQL) - Configurada en docker-compose.yml   No requiere Dockerfile propio; se define directamente en la orquestación.  YAML  db:
    # 1. Imagen oficial de MySQL 8.0[cite: 223].
    image: mysql:8.0                     
    
    # 2. Inyección de credenciales desde archivo .env[cite: 226, 289].
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASS}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASS}
      
    # 3. Mapeo de puertos para acceso externo[cite: 328, 504].
    ports:
      - "${DB_PORT:-3306}:3306"          
      
    # 4. Persistencia y script de inicialización automática[cite: 235, 297].
    volumes:
      - db-data:/var/lib/mysql           
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql  
      
    # 5. Prueba de vida para sincronizar con el Backend[cite: 237, 300].
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_ROOT_PASS}"]
[!NOTE]
Resumen del Flujo de Trabajo
Este sistema automatizado elimina instalaciones manuales. Con un solo comando, Docker compila el Frontend, empaqueta el Backend, inicializa la Base de Datos con sus tablas y conecta todo de forma segura bajo una red aislada.  
