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
    
    Client -->|"Carga la Interfaz Gráfica<br/>http://localhost:3000"| Frontend
    Client -->|"Envía y Recibe Datos (API)<br/>http://localhost:5000"| Backend
    Backend -->|"Consultas SQL<br/>jdbc:mysql://db:3306"| DB
    DB <-->|"Guarda la información físicamente"| Volumen
