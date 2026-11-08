# WebSocket Setup Guide

## 📡 Guía Completa de Configuración WebSocket

Esta guía detalla paso a paso la configuración del sistema de WebSocket para eventos en tiempo real en FUNA FENIX, utilizando Laravel Broadcasting, Soketi y Laravel Echo.

---

## 🎯 Arquitectura WebSocket

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│                 │    │                 │    │                 │
│   Frontend      │◄──►│   Soketi        │◄──►│   Laravel       │
│   (Laravel Echo)│    │   WebSocket     │    │   Broadcasting  │
│                 │    │   Server        │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         ▲                        ▲                        ▲
         │                        │                        │
         │                        │                        │
    Port 5173              Port 6001               Port 8000
    React Client          Soketi Server          Laravel API
```

---

## 🔧 Paso 1: Configuración del Backend Laravel

### 1.1 Instalación de Dependencias

```bash
# Navegar al directorio del backend
cd backend/src

# Instalar Pusher PHP SDK
composer require pusher/pusher-php-server

# Publicar configuración de broadcasting
php artisan vendor:publish --provider="Illuminate\Broadcasting\BroadcastServiceProvider"
```

### 1.2 Configuración del Archivo .env

```env
# Broadcasting Configuration
BROADCAST_DRIVER=pusher

# Pusher/Soketi Configuration
PUSHER_APP_ID=local-app
PUSHER_APP_KEY=local-key
PUSHER_APP_SECRET=local-secret
PUSHER_HOST=127.0.0.1
PUSHER_PORT=6001
PUSHER_SCHEME=http

# Queue Configuration (Requerido para Broadcasting)
QUEUE_CONNECTION=database
```

### 1.3 Configuración de Broadcasting

**Archivo: `config/broadcasting.php`**

```php
<?php

return [
    'default' => env('BROADCAST_DRIVER', 'null'),

    'connections' => [
        'pusher' => [
            'driver' => 'pusher',
            'key' => env('PUSHER_APP_KEY'),
            'secret' => env('PUSHER_APP_SECRET'),
            'app_id' => env('PUSHER_APP_ID'),
            'options' => [
                'host' => env('PUSHER_HOST', '127.0.0.1'),
                'port' => env('PUSHER_PORT', 6001),
                'scheme' => env('PUSHER_SCHEME', 'http'),
                'encrypted' => false,
                'useTLS' => false,
                'curl_options' => [
                    CURLOPT_SSL_VERIFYHOST => 0,
                    CURLOPT_SSL_VERIFYPEER => 0,
                ]
            ],
            'client_options' => [
                'verify' => false
            ],
        ],

        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
        ],

        'log' => [
            'driver' => 'log',
        ],

        'null' => [
            'driver' => 'null',
        ],
    ],
];
```

### 1.4 Configuración de Colas

**Crear tabla de jobs:**
```bash
php artisan queue:table
php artisan migrate
```

**Configuración en `config/queue.php`:**
```php
'default' => env('QUEUE_CONNECTION', 'database'),

'connections' => [
    'database' => [
        'driver' => 'database',
        'table' => 'jobs',
        'queue' => 'default',
        'retry_after' => 90,
        'after_commit' => false,
    ],
],
```

---

## 🔥 Paso 2: Configuración del Servidor Soketi

### 2.1 Instalación de Soketi

```bash
# Instalar globalmente
npm install -g @soketi/soketi

# O instalar localmente en el proyecto
npm install @soketi/soketi
```

### 2.2 Archivo de Configuración de Soketi

**Archivo: `soketi-config.json`**

```json
{
    "host": "0.0.0.0",
    "port": 6001,
    "metrics": {
        "enabled": true,
        "host": "0.0.0.0",
        "port": 9601
    },
    "debug": true,
    "database": {
        "redis": {
            "host": "127.0.0.1",
            "port": 6379,
            "keyPrefix": "soketi:"
        }
    },
    "rateLimiter": {
        "driver": "local"
    },
    "httpApi": {
        "acceptTraffic": {
            "memoryThreshold": 85
        }
    },
    "pusher": {
        "host": "127.0.0.1",
        "port": 6001,
        "key": "local-key",
        "secret": "local-secret",
        "app_id": "local-app",
        "useTLS": false
    },
    "ssl": {
        "enabled": false
    },
    "appManager": {
        "driver": "array",
        "array": {
            "apps": [
                {
                    "id": "local-app",
                    "key": "local-key",
                    "secret": "local-secret",
                    "maxConnections": 100,
                    "enableStats": true,
                    "enableClientMessages": true,
                    "maxBackendEventsPerSecond": 100,
                    "maxClientEventsPerSecond": 100,
                    "maxReadRequestsPerSecond": 100
                }
            ]
        }
    }
}
```

### 2.3 Script de Inicio de Soketi

```bash
#!/bin/bash
# Iniciar Soketi con configuración personalizada
npx soketi start \
    --config=soketi-config.json \
    --port=6001 \
    --metrics-server-port=9601 \
    --debug
```

---

## 📢 Paso 3: Implementación de Eventos

### 3.1 Estructura Base de Eventos

**Comando para crear evento:**
```bash
php artisan make:event FeedbackCreated
```

**Estructura del Evento:**

```php
<?php

namespace App\Events;

use App\Models\Feedback;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class FeedbackCreated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public Feedback $feedback;

    /**
     * Create a new event instance.
     */
    public function __construct(Feedback $feedback)
    {
        $this->feedback = $feedback;
    }

    /**
     * Get the channels the event should broadcast on.
     */
    public function broadcastOn(): array
    {
        return [
            new Channel('feedback-updates'),
        ];
    }

    /**
     * Get the data to broadcast.
     */
    public function broadcastWith(): array
    {
        return [
            'feedback' => $this->feedback->toArray()
        ];
    }

    /**
     * The event's broadcast name.
     */
    public function broadcastAs(): string
    {
        return 'feedback.created';
    }
}
```

### 3.2 Eventos Implementados

#### Eventos de Feedback:

1. **FeedbackCreated.php** - Feedback creado
2. **FeedbackUpdated.php** - Feedback actualizado  
3. **FeedbackDeleted.php** - Feedback eliminado

#### Eventos de Team Member:

1. **TeamMemberCreated.php** - Miembro creado
2. **TeamMemberUpdated.php** - Miembro actualizado
3. **TeamMemberDeleted.php** - Miembro eliminado

### 3.3 Configuración de Canales

**Archivo: `routes/channels.php`**

```php
<?php

use Illuminate\Support\Facades\Broadcast;

/*
|--------------------------------------------------------------------------
| Broadcast Channels
|--------------------------------------------------------------------------
*/

// Canal público para actualizaciones de feedback
Broadcast::channel('feedback-updates', function () {
    return true; // Canal público
});

// Canal público para actualizaciones de equipo
Broadcast::channel('team-updates', function () {
    return true; // Canal público
});

// Ejemplo de canal privado (requiere autenticación)
Broadcast::channel('user.{id}', function ($user, $id) {
    return (int) $user->id === (int) $id;
});
```

### 3.4 Disparar Eventos en Repositorios

**Archivo: `app/Repositories/FeedbackRepository.php`**

```php
<?php

namespace App\Repositories;

use App\Events\FeedbackCreated;
use App\Events\FeedbackUpdated;
use App\Events\FeedbackDeleted;
use App\Models\Feedback;

class FeedbackRepository
{
    public function create(array $data): Feedback
    {
        $feedback = Feedback::create($data);
        
        // Disparar evento de broadcasting
        event(new FeedbackCreated($feedback));
        
        return $feedback;
    }

    public function update(int $id, array $data): Feedback
    {
        $feedback = Feedback::findOrFail($id);
        $feedback->update($data);
        
        // Disparar evento de broadcasting
        event(new FeedbackUpdated($feedback));
        
        return $feedback;
    }

    public function delete(int $id): bool
    {
        $feedback = Feedback::findOrFail($id);
        $result = $feedback->delete();
        
        if ($result) {
            // Disparar evento de broadcasting
            event(new FeedbackDeleted($feedback));
        }
        
        return $result;
    }
}
```

---

## 🌐 Paso 4: Configuración del Frontend

### 4.1 Instalación de Dependencias

```bash
# Navegar al directorio del frontend
cd frontend

# Instalar Laravel Echo y Pusher.js
npm install laravel-echo pusher-js
```

### 4.2 Configuración de Laravel Echo

**Archivo: `src/api/echo.js`**

```javascript
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

// Configurar Pusher globalmente
window.Pusher = Pusher;

// Configuración de Echo para Soketi
const echo = new Echo({
    broadcaster: 'pusher',
    key: 'local-key',
    wsHost: 'localhost',
    wsPort: 6001,
    wssPort: 6001,
    forceTLS: false,
    encrypted: false,
    disableStats: true,
    enabledTransports: ['ws', 'wss'],
    cluster: false,
    authEndpoint: 'http://localhost:8000/broadcasting/auth',
    auth: {
        headers: {
            Authorization: `Bearer ${localStorage.getItem('auth_token')}`,
        },
    },
});

// Logging para debugging
echo.connector.pusher.connection.bind('connected', () => {
    console.log('✅ WebSocket conectado exitosamente');
});

echo.connector.pusher.connection.bind('disconnected', () => {
    console.log('❌ WebSocket desconectado');
});

echo.connector.pusher.connection.bind('error', (error) => {
    console.error('🔴 Error de WebSocket:', error);
});

export default echo;
```

### 4.3 Integración en Componentes React

**Archivo: `src/components/MainBoard.jsx`**

```javascript
import React, { useState, useEffect } from 'react';
import echo from '../api/echo';
import { getFeedbacks } from '../api/apiService';

const MainBoard = () => {
    const [feedbacks, setFeedbacks] = useState([]);
    const [loading, setLoading] = useState(true);

    // Cargar feedbacks iniciales
    useEffect(() => {
        const loadFeedbacks = async () => {
            try {
                const data = await getFeedbacks();
                setFeedbacks(data);
            } catch (error) {
                console.error('Error cargando feedbacks:', error);
            } finally {
                setLoading(false);
            }
        };

        loadFeedbacks();
    }, []);

    // Configurar listeners de WebSocket
    useEffect(() => {
        console.log('🔗 Configurando listeners de WebSocket...');

        // Canal de actualizaciones de feedback
        const feedbackChannel = echo.channel('feedback-updates');

        // Escuchar evento de feedback creado
        feedbackChannel.listen('feedback.created', (event) => {
            console.log('📝 Nuevo feedback recibido:', event.feedback);
            setFeedbacks(prevFeedbacks => [...prevFeedbacks, event.feedback]);
        });

        // Escuchar evento de feedback actualizado
        feedbackChannel.listen('feedback.updated', (event) => {
            console.log('✏️ Feedback actualizado:', event.feedback);
            setFeedbacks(prevFeedbacks => 
                prevFeedbacks.map(feedback => 
                    feedback.id === event.feedback.id ? event.feedback : feedback
                )
            );
        });

        // Escuchar evento de feedback eliminado
        feedbackChannel.listen('feedback.deleted', (event) => {
            console.log('🗑️ Feedback eliminado:', event.feedback);
            setFeedbacks(prevFeedbacks => 
                prevFeedbacks.filter(feedback => feedback.id !== event.feedback.id)
            );
        });

        // Cleanup al desmontar componente
        return () => {
            console.log('🧹 Limpiando listeners de WebSocket...');
            echo.leaveChannel('feedback-updates');
        };
    }, []);

    // Canal de actualizaciones de equipo
    useEffect(() => {
        const teamChannel = echo.channel('team-updates');

        teamChannel.listen('team-member.created', (event) => {
            console.log('👥 Nuevo miembro del equipo:', event.user);
            // Actualizar lista de usuarios si es necesario
        });

        teamChannel.listen('team-member.updated', (event) => {
            console.log('👤 Miembro actualizado:', event.user);
            // Actualizar datos del usuario
        });

        teamChannel.listen('team-member.deleted', (event) => {
            console.log('❌ Miembro eliminado:', event.user);
            // Remover usuario de la lista
        });

        return () => {
            echo.leaveChannel('team-updates');
        };
    }, []);

    if (loading) {
        return <div>Cargando feedbacks...</div>;
    }

    return (
        <div className="main-board">
            <h2>Feedbacks del Equipo</h2>
            <div className="feedback-grid">
                {feedbacks.map(feedback => (
                    <div key={feedback.id} className="feedback-card">
                        <h3>{feedback.title}</h3>
                        <p>{feedback.text}</p>
                        <small>Categoría: {feedback.category}</small>
                    </div>
                ))}
            </div>
        </div>
    );
};

export default MainBoard;
```

### 4.4 Hook Personalizado para WebSocket

**Archivo: `src/hooks/useWebSocket.js`**

```javascript
import { useEffect, useState } from 'react';
import echo from '../api/echo';

export const useWebSocket = (channel, events = {}) => {
    const [connected, setConnected] = useState(false);
    const [error, setError] = useState(null);

    useEffect(() => {
        // Configurar estado de conexión
        const onConnected = () => setConnected(true);
        const onDisconnected = () => setConnected(false);
        const onError = (error) => setError(error);

        echo.connector.pusher.connection.bind('connected', onConnected);
        echo.connector.pusher.connection.bind('disconnected', onDisconnected);
        echo.connector.pusher.connection.bind('error', onError);

        // Configurar canal y eventos
        const wsChannel = echo.channel(channel);
        
        Object.entries(events).forEach(([eventName, handler]) => {
            wsChannel.listen(eventName, handler);
        });

        // Cleanup
        return () => {
            echo.connector.pusher.connection.unbind('connected', onConnected);
            echo.connector.pusher.connection.unbind('disconnected', onDisconnected);
            echo.connector.pusher.connection.unbind('error', onError);
            echo.leaveChannel(channel);
        };
    }, [channel, events]);

    return { connected, error };
};

// Uso del hook
const MyComponent = () => {
    const { connected, error } = useWebSocket('feedback-updates', {
        'feedback.created': (event) => console.log('Nuevo feedback:', event),
        'feedback.updated': (event) => console.log('Feedback actualizado:', event),
        'feedback.deleted': (event) => console.log('Feedback eliminado:', event),
    });

    return (
        <div>
            <p>Estado: {connected ? '🟢 Conectado' : '🔴 Desconectado'}</p>
            {error && <p>Error: {error.message}</p>}
        </div>
    );
};
```

---

## 🐳 Paso 5: Automatización con Docker

### 5.1 Script de Inicio Automatizado

**Archivo: `backend/start.sh`**

```bash
#!/bin/bash
set -e

echo "🔥 Iniciando servicios FUNA FENIX..."

# Verificar dependencias
check_dependencies() {
    if ! command -v composer &> /dev/null; then
        echo "❌ Composer no encontrado"
        exit 1
    fi
    
    if ! command -v php &> /dev/null; then
        echo "❌ PHP no encontrado"
        exit 1
    fi
    
    if ! command -v node &> /dev/null; then
        echo "❌ Node.js no encontrado"
        exit 1
    fi
}

# Configurar aplicación Laravel
setup_laravel() {
    echo "📦 Instalando dependencias de Composer..."
    composer install --no-dev --optimize-autoloader

    echo "🔧 Configurando aplicación..."
    cp .env.example .env 2>/dev/null || echo "📁 .env ya existe"
    php artisan key:generate --force
    php artisan config:cache
    php artisan route:cache

    echo "🗄️ Ejecutando migraciones..."
    php artisan migrate --force

    echo "🌱 Ejecutando seeders..."
    php artisan db:seed --force
}

# Iniciar servicios
start_services() {
    echo "🚀 Iniciando Queue Worker..."
    php artisan queue:work --daemon &
    QUEUE_PID=$!

    echo "📡 Iniciando Soketi WebSocket Server..."
    npx soketi start \
        --config=/var/www/html/soketi-config.json \
        --port=6001 \
        --metrics-server-port=9601 \
        --debug &
    SOKETI_PID=$!

    # Esperar a que Soketi esté listo
    sleep 3
    
    echo "🌐 Iniciando Laravel Server..."
    php artisan serve --host=0.0.0.0 --port=8000 &
    LARAVEL_PID=$!

    # Función de limpieza
    cleanup() {
        echo "🛑 Deteniendo servicios..."
        kill $QUEUE_PID $SOKETI_PID $LARAVEL_PID 2>/dev/null || true
        exit 0
    }
    trap cleanup SIGINT SIGTERM

    echo "✅ Todos los servicios iniciados correctamente"
    echo "🌐 Frontend: http://localhost:5173"
    echo "🔗 Backend API: http://localhost:8000"
    echo "📡 WebSocket: http://localhost:6001"
    echo "📊 Soketi Metrics: http://localhost:9601"

    # Mantener el script corriendo
    wait
}

# Ejecutar funciones principales
check_dependencies
setup_laravel
start_services
```

### 5.2 Dockerfile Actualizado

**Archivo: `backend/dockerfile`**

```dockerfile
FROM php:8.2-fpm

# Instalar dependencias del sistema
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    default-mysql-client

# Limpiar cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Instalar extensiones PHP
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Instalar Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Instalar Node.js LTS
RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs

# Instalar Soketi globalmente
RUN npm install -g @soketi/soketi

# Configurar directorio de trabajo
WORKDIR /var/www/html

# Copiar archivos de la aplicación
COPY src/ .
COPY start.sh .

# Dar permisos de ejecución
RUN chmod +x start.sh

# Dar permisos apropiados
RUN chown -R www-data:www-data /var/www/html
RUN chmod -R 755 /var/www/html/storage
RUN chmod -R 755 /var/www/html/bootstrap/cache

# Exponer puertos
EXPOSE 8000 6001 9601

# Comando por defecto
CMD ["./start.sh"]
```

### 5.3 Docker Compose Actualizado

**Archivo: `docker-compose.yml`**

```yaml
version: '3.8'

services:
  backend:
    build: 
      context: ./backend
      dockerfile: dockerfile
    container_name: fenix-backend
    ports:
      - "8000:8000"   # Laravel API
      - "6001:6001"   # Soketi WebSocket
      - "9601:9601"   # Soketi Metrics
    environment:
      - DB_HOST=mysql
      - DB_DATABASE=fenix_feedback
      - DB_USERNAME=fenix_user
      - DB_PASSWORD=fenix_password
      - BROADCAST_DRIVER=pusher
      - QUEUE_CONNECTION=database
      - PUSHER_APP_ID=local-app
      - PUSHER_APP_KEY=local-key
      - PUSHER_APP_SECRET=local-secret
      - PUSHER_HOST=127.0.0.1
      - PUSHER_PORT=6001
      - PUSHER_SCHEME=http
    depends_on:
      mysql:
        condition: service_healthy
    command: ["/var/www/html/start.sh"]
    volumes:
      - ./backend/src:/var/www/html
      - ./soketi-config.json:/var/www/html/soketi-config.json
    networks:
      - fenix-network

  frontend:
    build:
      context: ./frontend
      dockerfile: dockerfile
    container_name: fenix-frontend
    ports:
      - "5173:5173"   # Vite Dev Server
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - CHOKIDAR_USEPOLLING=true
    networks:
      - fenix-network

  mysql:
    image: mysql:8.0
    container_name: fenix-mysql
    restart: always
    environment:
      MYSQL_DATABASE: fenix_feedback
      MYSQL_USER: fenix_user
      MYSQL_PASSWORD: fenix_password
      MYSQL_ROOT_PASSWORD: root_password
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 5s
      retries: 10
      interval: 10s
      start_period: 30s
    networks:
      - fenix-network

volumes:
  mysql_data:
    driver: local

networks:
  fenix-network:
    driver: bridge
```

---

## 🔍 Paso 6: Debugging y Solución de Problemas

### 6.1 Verificación de Conexiones

#### Verificar Soketi está corriendo:
```bash
# Verificar puerto 6001
curl http://localhost:6001/

# Verificar métricas
curl http://localhost:9601/metrics
```

#### Verificar Queue Worker:
```bash
# Ver logs del queue worker
docker compose logs backend | grep queue

# Verificar jobs en la base de datos
docker compose exec backend php artisan queue:failed
```

#### Verificar Broadcasting en Laravel:
```bash
# Test de evento manual
docker compose exec backend php artisan tinker
# En tinker: event(new App\Events\FeedbackCreated(\App\Models\Feedback::first()));
```

### 6.2 Debugging Frontend

#### Console del navegador:
```javascript
// Verificar estado de conexión Echo
echo.connector.pusher.connection.state

// Verificar canales suscritos
echo.connector.channels

// Debug de eventos
echo.channel('feedback-updates').bind('pusher:subscription_succeeded', () => {
    console.log('✅ Suscrito a canal feedback-updates');
});
```

#### Network Tab del navegador:
- Buscar conexión WebSocket a `ws://localhost:6001`
- Verificar upgrade de HTTP a WebSocket
- Monitorear mensajes enviados y recibidos

### 6.3 Logs y Monitoreo

#### Logs de Laravel:
```bash
# Ver logs del backend
docker compose logs backend --follow

# Logs específicos de Laravel
tail -f backend/src/storage/logs/laravel.log
```

#### Logs de Soketi:
```bash
# Los logs aparecen en la salida del contenedor backend
docker compose logs backend | grep soketi
```

#### Logs de MySQL:
```bash
# Logs de base de datos
docker compose logs mysql

# Conectar a MySQL para debugging
docker compose exec mysql mysql -u fenix_user -pfenix_password fenix_feedback
```

### 6.4 Problemas Comunes y Soluciones

#### 🔴 **Problema: WebSocket no conecta**

**Síntomas:**
- Console error: "WebSocket connection to 'ws://localhost:6001' failed"
- Echo state muestra "disconnected"

**Soluciones:**
```bash
# 1. Verificar que Soketi esté corriendo
curl http://localhost:6001/

# 2. Verificar configuración de puertos en docker-compose
docker compose ps

# 3. Revisar configuración de echo.js
echo.connector.pusher.config
```

#### 🔴 **Problema: Eventos no se reciben**

**Síntomas:**
- Echo conecta pero no recibe eventos
- Backend funciona pero frontend no se actualiza

**Soluciones:**
```bash
# 1. Verificar que Queue Worker esté corriendo
docker compose exec backend php artisan queue:work --once

# 2. Verificar tabla de jobs
docker compose exec backend php artisan queue:failed

# 3. Test manual de evento
docker compose exec backend php artisan tinker
```

#### 🔴 **Problema: Authentication para canales privados**

**Síntomas:**
- Error 403 en canales privados
- "Unable to retrieve auth string from auth endpoint"

**Soluciones:**
```javascript
// Configurar headers de autorización en echo.js
auth: {
    headers: {
        Authorization: `Bearer ${localStorage.getItem('auth_token')}`,
        Accept: 'application/json',
        'Content-Type': 'application/json',
    },
},
```

---

## 📊 Paso 7: Testing y Validación

### 7.1 Test Manual de WebSocket

```bash
# Terminal 1: Monitorear logs
docker compose logs backend --follow

# Terminal 2: Crear feedback via API
curl -X POST http://localhost:8000/api/feedbacks \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "target_id": 1,
    "category": "achievements",
    "title": "Test WebSocket",
    "text": "Este feedback debería aparecer en tiempo real"
  }'

# Verificar en frontend que el feedback aparezca automáticamente
```

### 7.2 Test de Carga de WebSocket

```javascript
// Script para test de múltiples conexiones
const connections = [];

for (let i = 0; i < 10; i++) {
    const testEcho = new Echo({
        broadcaster: 'pusher',
        key: 'local-key',
        wsHost: 'localhost',
        wsPort: 6001,
        forceTLS: false,
        encrypted: false,
    });
    
    testEcho.channel('feedback-updates')
        .listen('feedback.created', (e) => {
            console.log(`Conexión ${i} recibió evento:`, e);
        });
    
    connections.push(testEcho);
}

// Limpiar conexiones después del test
// connections.forEach(conn => conn.disconnect());
```

### 7.3 Test de Resilencia

```bash
# Test de reconexión automática
# 1. Iniciar sistema normalmente
docker compose up

# 2. Detener solo Soketi
docker compose exec backend pkill soketi

# 3. Verificar que frontend detecte desconexión
# 4. Reiniciar Soketi
docker compose exec backend npx soketi start --config=soketi-config.json --port=6001 &

# 5. Verificar reconexión automática en frontend
```

---

## 📈 Paso 8: Optimización y Producción

### 8.1 Configuración para Producción

**Variables de entorno para producción:**
```env
# App
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-domain.com

# Broadcasting
BROADCAST_DRIVER=pusher
PUSHER_APP_ID=your-prod-app-id
PUSHER_APP_KEY=your-prod-key
PUSHER_APP_SECRET=your-prod-secret
PUSHER_HOST=your-soketi-host.com
PUSHER_PORT=443
PUSHER_SCHEME=https

# SSL/TLS
PUSHER_USE_TLS=true
SOKETI_SSL_ENABLED=true
```

**Configuración Soketi para producción:**
```json
{
    "host": "0.0.0.0",
    "port": 6001,
    "ssl": {
        "enabled": true,
        "certPath": "/path/to/cert.pem",
        "keyPath": "/path/to/key.pem"
    },
    "database": {
        "redis": {
            "host": "redis-server",
            "port": 6379,
            "keyPrefix": "soketi:"
        }
    },
    "appManager": {
        "driver": "mysql",
        "mysql": {
            "host": "mysql-server",
            "port": 3306,
            "username": "soketi_user",
            "password": "secure_password",
            "database": "soketi"
        }
    }
}
```

### 8.2 Optimizaciones de Rendimiento

#### Backend (Laravel):
```php
// config/broadcasting.php - Optimizaciones
'pusher' => [
    'driver' => 'pusher',
    'key' => env('PUSHER_APP_KEY'),
    'secret' => env('PUSHER_APP_SECRET'),
    'app_id' => env('PUSHER_APP_ID'),
    'options' => [
        'cluster' => env('PUSHER_APP_CLUSTER'),
        'useTLS' => true,
        'timeout' => 30,
        'curl_options' => [
            CURLOPT_CONNECTTIMEOUT => 10,
            CURLOPT_TIMEOUT => 30,
        ]
    ],
],

// Queue configuration para alto volumen
'redis' => [
    'driver' => 'redis',
    'connection' => 'default',
    'queue' => 'broadcasting',
    'retry_after' => 90,
    'block_for' => null,
],
```

#### Frontend (React):
```javascript
// Configuración optimizada para producción
const echo = new Echo({
    broadcaster: 'pusher',
    key: process.env.REACT_APP_PUSHER_KEY,
    cluster: process.env.REACT_APP_PUSHER_CLUSTER,
    forceTLS: true,
    encrypted: true,
    enableStats: false,
    
    // Configuración de reconexión
    enabledTransports: ['ws', 'wss'],
    activityTimeout: 30000,
    pongTimeout: 6000,
    unavailableTimeout: 10000,
});

// Manejo de errores mejorado
echo.connector.pusher.connection.bind('error', (error) => {
    console.error('WebSocket error:', error);
    // Implementar notificación al usuario
});
```

### 8.3 Monitoreo y Alertas

```bash
# Script de monitoreo de salud del WebSocket
#!/bin/bash

SOKETI_URL="http://localhost:6001"
HEALTH_CHECK=$(curl -s -o /dev/null -w "%{http_code}" $SOKETI_URL)

if [ $HEALTH_CHECK != "200" ]; then
    echo "🚨 ALERTA: Soketi no responde correctamente"
    # Enviar notificación (email, Slack, etc.)
    # Intentar reiniciar servicio automáticamente
else
    echo "✅ Soketi funcionando correctamente"
fi

# Verificar Queue Worker
QUEUE_FAILED=$(docker compose exec backend php artisan queue:failed --format=json | jq length)
if [ $QUEUE_FAILED -gt 0 ]; then
    echo "🚨 ALERTA: $QUEUE_FAILED jobs fallidos en cola"
fi
```

---

## 📚 Recursos Adicionales

### Documentación Oficial:
- [Laravel Broadcasting](https://laravel.com/docs/broadcasting)
- [Laravel Echo](https://github.com/laravel/echo)
- [Soketi Documentation](https://docs.soketi.app/)
- [Pusher Protocol](https://pusher.com/docs/channels/library_auth_reference/pusher-websockets-protocol)

### Herramientas de Debug:
- [Pusher Debug Console](https://dashboard.pusher.com/apps/debug_console)
- [WebSocket King](https://websocketking.com/) - Cliente WebSocket para testing
- [Postman WebSocket](https://www.postman.com/) - Testing de WebSocket

### Alternativas a Soketi:
- **Laravel Reverb** - Servidor WebSocket nativo de Laravel 11
- **Pusher Channels** - Servicio SaaS de Pusher
- **AWS API Gateway WebSocket** - Para AWS deployments
- **Redis + NodeJS** - Implementación custom

---

## ⚡ Inicio Rápido

### Comandos Esenciales:

```bash
# Iniciar todo el sistema
docker compose up --build

# Reiniciar solo WebSocket
docker compose restart backend

# Ver logs en tiempo real
docker compose logs backend --follow

# Test manual de eventos
docker compose exec backend php artisan tinker
# En tinker: event(new App\Events\FeedbackCreated(\App\Models\Feedback::first()));

# Verificar estado de servicios
curl http://localhost:6001/        # Soketi
curl http://localhost:8000/api    # Laravel API
curl http://localhost:9601/metrics # Métricas Soketi
```

### Checklist de Verificación:

- ✅ **Soketi corriendo en puerto 6001**
- ✅ **Queue Worker procesando jobs**
- ✅ **Laravel API respondiendo**
- ✅ **Frontend conectando a WebSocket**
- ✅ **Eventos disparándose desde repositorios**
- ✅ **Echo recibiendo eventos en frontend**

---

## 🎯 Conclusión

Esta configuración proporciona un sistema de WebSocket robusto y escalable para la aplicación FUNA FENIX. El stack utilizado (Laravel Broadcasting + Soketi + Laravel Echo) es una alternativa moderna y confiable a Pusher, con control completo sobre la infraestructura.

**Características implementadas:**
- ✅ **Eventos en tiempo real bidireccionales**
- ✅ **Reconexión automática**
- ✅ **Canales públicos y privados**
- ✅ **Autenticación de canales**
- ✅ **Escalabilidad horizontal**
- ✅ **Monitoreo y métricas**
- ✅ **Configuración para desarrollo y producción**

El sistema está listo para manejar actualizaciones en tiempo real de feedback del equipo, garantizando una experiencia fluida y colaborativa para todos los usuarios.
