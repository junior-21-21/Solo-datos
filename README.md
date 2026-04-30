# 🐾 Sistema de Inventarios - Clínica Veterinaria Petyzoos

Sistema web de gestión de inventarios para la Clínica Veterinaria Petyzoos, desarrollado con **Angular 21** (frontend) y **Spring Boot 4** (backend) con base de datos **MySQL 8**.

## 📋 Tabla de Contenidos

- [Descripción](#descripción)
- [Requisitos](#requisitos)
- [Instalación con Docker](#instalación-con-docker)
- [Endpoints de API](#endpoints-de-api)
- [Estructura del Proyecto](#estructura-del-proyecto)
- [Troubleshooting](#troubleshooting)

## Descripción

Sistema completo de gestión de inventarios que incluye:
- **Dashboard** con estadísticas en tiempo real
- **Gestión de productos** con código de barras y control de stock
- **Gestión de categorías, proveedores y clientes**
- **Registro de compras** (ingresos) con lotes y fechas de vencimiento
- **Registro de salidas** (ventas/consumo interno)
- **Kardex** para trazabilidad de movimientos
- **Control de lotes** con alertas de vencimiento
- **Autenticación JWT** con roles (Administrador/Veterinario)

## Requisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) v20.10+
- [Docker Compose](https://docs.docker.com/compose/) v2.0+
- 4GB RAM mínimo disponible

## Instalación con Docker

### 1. Clonar el repositorio
```bash
git clone <url-del-repositorio>
cd Sistema-de-inventarios
```

### 2. Configurar variables de entorno
```bash
cp .env.example .env
# Editar .env con valores reales
```

### 3. Construir y ejecutar
```bash
docker-compose up -d --build
```

### 4. Verificar servicios
```bash
docker-compose ps
```

### 5. Acceder a la aplicación
| Servicio | URL |
|----------|-----|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:5000/api |
| MySQL | localhost:3306 |

### Credenciales por defecto
| Usuario | Contraseña | Rol |
|---------|-----------|-----|
| admin | admin123 | Administrador |
| veterinario | vet123 | Veterinario |

## Endpoints de API

### Autenticación
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/auth/login` | Iniciar sesión |

### Productos
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/productos` | Listar todos los productos |
| GET | `/api/productos/{codigo}` | Obtener producto por código |
| GET | `/api/productos/buscar?nombre=...` | Buscar por nombre |
| GET | `/api/productos/stock-critico` | Productos con stock bajo |
| POST | `/api/productos` | Crear producto |
| PUT | `/api/productos/{codigo}` | Actualizar producto |
| DELETE | `/api/productos/{codigo}` | Eliminar producto |

### Categorías
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/categorias` | Listar categorías |
| POST | `/api/categorias` | Crear categoría |
| PUT | `/api/categorias/{id}` | Actualizar categoría |
| DELETE | `/api/categorias/{id}` | Eliminar categoría |

### Compras (Ingresos)
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/compras` | Listar compras |
| POST | `/api/compras` | Registrar compra |

### Salidas (Ventas)
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/salidas` | Listar salidas |
| POST | `/api/salidas` | Registrar salida |

### Kardex
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/kardex` | Listar movimientos |
| GET | `/api/kardex/producto/{codigo}` | Kardex por producto |
| GET | `/api/kardex/filtrar?desde=...&hasta=...` | Filtrar por fecha |

### Otros
| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/dashboard` | Estadísticas |
| GET | `/api/proveedores` | Listar proveedores |
| GET | `/api/clientes` | Listar clientes |
| GET | `/api/usuarios` | Listar usuarios |
| GET | `/api/roles` | Listar roles |
| GET | `/api/lotes` | Listar lotes |

## Estructura del Proyecto

```
Sistema-de-inventarios/
├── frontend-inventario/     # Angular 21 + Material
│   ├── Dockerfile           # Multi-stage build
│   ├── nginx.conf           # Configuración Nginx
│   ├── .dockerignore
│   └── src/
├── backend/                 # Spring Boot 4 + JPA
│   ├── Dockerfile           # Multi-stage build + HEALTHCHECK
│   ├── .dockerignore
│   └── src/
├── db/
│   └── init.sql             # Script de inicialización MySQL
├── docker-compose.yml       # Orquestación de servicios
├── .env                     # Variables de entorno (NO commitear)
├── .env.example             # Ejemplo de variables
├── .gitignore
├── .dockerignore
└── README.md
```

## Troubleshooting

### Error: "Connection refused" al conectar backend con BD
```bash
# Verificar que MySQL está corriendo
docker-compose ps
# Ver logs de MySQL
docker-compose logs db
# Verificar conectividad desde backend
docker exec petyzoos-backend ping db
```

### Error: Frontend muestra página en blanco
```bash
# Ver logs del frontend
docker-compose logs frontend
# Verificar que el build de Angular fue exitoso
docker exec petyzoos-frontend ls /usr/share/nginx/html
```

### Error: "Unauthorized" en la API
```bash
# Verificar que el backend está sano
docker inspect petyzoos-backend | grep -A 5 Health
# Ver logs del backend para errores JWT
docker-compose logs backend | grep -i error
```

### Resetear todo (datos incluidos)
```bash
docker-compose down -v    # -v elimina volúmenes
docker-compose up -d --build
```

### Ver logs en tiempo real
```bash
docker-compose logs -f    # Todos los servicios
docker-compose logs -f backend  # Solo backend
```
"# Solo-datos" 
