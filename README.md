# RetroApp - Plataforma de Retrospectivas para Equipos de Desarrollo 🔥

[![Docker](https://img.shields.io/badge/Docker-Ready-blue.svg)](https://docs.docker.com/)
[![Laravel](https://img.shields.io/badge/Laravel-11-red.svg)](https://laravel.com/)
[![React](https://img.shields.io/badge/React-18-blue.svg)](https://reactjs.org/)
[![WebSocket](https://img.shields.io/badge/WebSocket-Soketi-green.svg)](https://soketi.app/)

Plataforma colaborativa en tiempo real diseñada para dinámicas de retroalimentación en equipos de desarrollo. Permite compartir feedback constructivo (logros, cualidades y oportunidades de mejora), gestionar miembros y sincronizar estados al instante vía WebSockets.

---

## 📋 Tabla de Contenidos

- [Características Principales](#-características-principales)
- [Arquitectura y Stack](#️-arquitectura-y-stack)
- [Inicio Rápido](#-inicio-rápido)
- [Configuración de Entorno](#️-configuración-de-entorno)
- [Estructura del Proyecto](#-estructura-del-proyecto)
- [API y WebSockets](#-api-y-websockets)
- [Comandos de Desarrollo](#-comandos-de-desarrollo)
- [Diagnóstico y Troubleshooting](#️-diagnóstico-y-troubleshooting)

---

## ✨ Características Principales

- **Sincronización en Tiempo Real**: Tablero interactivo tipo *post-it* impulsado por Soketi y Laravel Echo.
- **Categorización de Feedback**: Segmentación por logros, cualidades y áreas de crecimiento.
- **Control de Acceso Basado en Roles (RBAC)**: Autenticación con Laravel Sanctum y roles diferenciados (`admin`, `owner`, `member`).
- **Administración de Miembros**: CRUD completo de integrantes y visualización de feedback por persona.
- **Contenerización Total**: Entorno multi-contenedor orquestado con Docker Compose.

---

## 🏗️ Arquitectura y Stack

| Capa | Tecnologías | Propósito |
|---|---|---|
| **Frontend** | React 18, Vite, Tailwind CSS, Axios, Laravel Echo | SPA reactiva con actualización sin recarga |
| **Backend** | Laravel 11 (PHP 8.2), Laravel Sanctum | API RESTful y despacho de eventos en tiempo real |
| **Tiempo Real** | Soketi (Pusher Protocol), Database Queue Worker | Servidor WebSocket de baja latencia y colas |
| **Persistencia** | MySQL 8.0 | Almacenamiento relacional |
| **Infraestructura** | Docker Compose, Alpine Linux | Orquestación y paridad entre entornos |

```
Usuario → Frontend (React) ──[HTTP REST]──→ Backend (Laravel) ──→ DB (MySQL)
              ▲                                   │
              └──────[WebSocket]──── Soketi ◄─────┘ (Queue Worker / Events)
```

---

## 🚀 Inicio Rápido

### Prerrequisitos
- Docker Engine 20.10+ y Docker Compose 2.0+
- Puertos libres: `3306` (MySQL), `5173` (Vite), `6001` (Soketi), `8000` (Laravel)

### Despliegue en 2 Pasos

```bash
# 1. Clonar el repositorio y acceder
git clone https://github.com/tu-usuario/retro-app.git
cd retro-app

# 2. Levantar la suite de servicios
docker compose up --build
```

### Accesos y Credenciales por Defecto

| Servicio | URL / Host | Credenciales |
|---|---|---|
| **Frontend** | `http://localhost:5173` | — |
| **API Backend** | `http://localhost:8000` | — |
| **WebSocket** | `http://localhost:6001` | Key: `local-key` |
| **MySQL** | `localhost:3306` | User: `app_user` / DB: `retro_db` |
| **Login Admin** | Vía Frontend o `/api/auth/login` | `admin@admin.com` / `admin` |

---

## ⚙️ Configuración de Entorno

### Variables de Entorno Raíz (`.env`)
```bash
# Base de Datos
DB_ROOT_PASSWORD=root_secret
DB_DATABASE=retro_db
DB_USERNAME=app_user
DB_PASSWORD=app_secret

# WebSocket / Broadcasting
PUSHER_APP_ID=local-app
PUSHER_APP_KEY=local-key  
PUSHER_APP_SECRET=local-secret

# URLs de Servicios
FRONTEND_URL=http://localhost:5173
BACKEND_URL=http://localhost:8000
```

### Configuración WebSocket (`soketi-config.json`)
```json
{
  "debug": false,
  "port": 6001,
  "appManager": {
    "driver": "array",
    "array": [
      {
        "id": "local-app",
        "key": "local-key", 
        "secret": "local-secret",
        "maxConnections": 100,
        "enableClientMessages": true
      }
    ]
  },
  "cors": {
    "credentials": true,
    "origin": ["http://localhost:5173"],
    "methods": ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
  }
}
```

---

## 📁 Estructura del Proyecto

```
retro-app/
├── docker-compose.yml           # Orquestación de contenedores
├── soketi-config.json          # Reglas del servidor WebSocket
├── docs/                       # Guías de arquitectura, API y frontend
├── backend/                    # Core Laravel
│   ├── dockerfile
│   ├── start.sh                # Script de arranque (migraciones + workers)
│   └── src/
│       ├── app/
│       │   ├── Events/         # Eventos emitidos hacia Soketi
│       │   ├── Http/Controllers/
│       │   └── Models/
│       └── routes/
│           ├── api.php         # Endpoints REST
│           └── channels.php    # Autorización de canales privados
└── frontend/                   # SPA React + Vite
    ├── dockerfile
    └── src/
        ├── api/                # Cliente HTTP y configuración Echo
        ├── components/         # Tablero, modales y post-it notes
        └── App.jsx
```

---

## 🔌 API y WebSockets

### Endpoints Principales

| Método | Endpoint | Descripción | Acceso |
|---|---|---|---|
| `POST` | `/api/auth/login` | Autenticación y emisión de token | Público |
| `POST` | `/api/auth/logout` | Revocación de sesión | Autenticado |
| `GET` | `/api/feedbacks` | Lista feedbacks registrados | Autenticado |
| `POST` | `/api/feedbacks` | Crea una nueva nota | Autenticado |
| `DELETE` | `/api/feedbacks/{id}` | Elimina feedback | Owner / Admin |
| `GET\|POST` | `/api/team-members` | Gestión de miembros del equipo | Admin (`POST`) |

### Eventos en Tiempo Real

- **Canal `feedback-updates`**:
  - `feedback.created`: Inserción inmediata de nueva tarjeta.
  - `feedback.updated`: Modificación de contenido o estado.
  - `feedback.deleted`: Remoción de la nota en el tablero de todos los clientes.
- **Canal `team-updates`**:
  - `team-member.created` / `.updated` / `.deleted`: Sincronización del directorio de equipo.

---

## 🔧 Comandos de Desarrollo

```bash
# Ciclo de Vida Docker
docker compose up -d                 # Levantar en segundo plano
docker compose restart backend       # Reiniciar servicio backend
docker compose down -v               # Destruir contenedores y volúmenes

# Operaciones Backend (Laravel)
docker compose exec backend php artisan migrate --seed
docker compose exec backend php artisan queue:work
docker compose exec backend tail -f storage/logs/laravel.log

# Operaciones Frontend (React)
docker compose exec frontend npm install <paquete>
docker compose exec frontend npm run build

# Acceso a Base de Datos
docker compose exec database mysql -u app_user -p retro_db
```

---

## ⚠️ Diagnóstico y Troubleshooting

| Síntoma | Causa Probable | Solución |
|---|---|---|
| **Notas no sincronizan en vivo** | Soketi o Queue Worker detenidos | Verificar puertos con `curl http://localhost:6001/` y comprobar el proceso worker con `docker compose exec backend ps aux \| grep queue`. |
| **Error 401 / 403 en peticiones** | Dominio no autorizado en Sanctum / CORS | Revisar `SANCTUM_STATEFUL_DOMAINS` en backend y los orígenes en `soketi-config.json`. |
| **Fallo de conexión a base de datos** | Contenedor MySQL aún inicializándose | Esperar a que el healthcheck pase a `healthy` o reiniciar con `docker compose restart database`. |
| **Puerto en conflicto (8000/5173/6001)** | Proceso previo ocupando el socket | Identificar y liberar con `lsof -ti:<puerto> \| xargs kill -9`. |
