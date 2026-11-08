⚙️ Guía de Configuración - RetroApp Backend
Descripción General
Esta guía detalla las configuraciones necesarias para el funcionamiento del backend de RetroApp, incluyendo variables de entorno, broadcasting en tiempo real (Soketi), autenticación con Sanctum y configuraciones del framework Laravel 11.
Variables de Entorno (.env)
Archivo .env Principal (Backend)
# ========================================
# APLICACIÓN
# ========================================
APP_NAME="RetroApp Backend"
APP_ENV=local
APP_KEY=base64:GENERATED_APP_KEY_HERE
APP_DEBUG=true
APP_URL=http://localhost:8000

APP_LOCALE=es
APP_FALLBACK_LOCALE=en

# ========================================
# BASE DE DATOS (MySQL)
# ========================================
DB_CONNECTION=mysql
DB_HOST=database
DB_PORT=3306
DB_DATABASE=retro_db
DB_USERNAME=app_user
DB_PASSWORD=app_secret

# ========================================
# BROADCASTING & WEBSOCKETS (Soketi)
# ========================================
BROADCAST_DRIVER=pusher
PUSHER_APP_ID=local-app
PUSHER_APP_KEY=local-key
PUSHER_APP_SECRET=local-secret
PUSHER_HOST=127.0.0.1
PUSHER_PORT=6001
PUSHER_SCHEME=http

# ========================================
# COLAS, CACHÉ Y SESIÓN
# ========================================
QUEUE_CONNECTION=database
CACHE_STORE=database
SESSION_DRIVER=database
SESSION_LIFETIME=120

# ========================================
# AUTENTICACIÓN (Sanctum) & CORS
# ========================================
SANCTUM_STATEFUL_DOMAINS=localhost,localhost:5173,127.0.0.1,127.0.0.1:5173
FRONTEND_URL=http://localhost:5173

# ========================================
# LOGS
# ========================================
LOG_CHANNEL=stack
LOG_LEVEL=debug

Variables para Docker Compose (.env raíz)
DB_ROOT_PASSWORD=root_secret
DB_DATABASE=retro_db
DB_USERNAME=app_user
DB_PASSWORD=app_secret

PUSHER_APP_ID=local-app
PUSHER_APP_KEY=local-key
PUSHER_APP_SECRET=local-secret

FRONTEND_URL=http://localhost:5173
BACKEND_URL=http://localhost:8000

Broadcasting y WebSockets
Configuración Laravel (config/broadcasting.php)
<?php

return [
    'default' => env('BROADCAST_DRIVER', 'pusher'),

    'connections' => [
        'pusher' => [
            'driver' => 'pusher',
            'key' => env('PUSHER_APP_KEY'),
            'secret' => env('PUSHER_APP_SECRET'),
            'app_id' => env('PUSHER_APP_ID'),
            'options' => [
                'host' => env('PUSHER_HOST', '127.0.0.1'),
                'port' => (int) env('PUSHER_PORT', 6001),
                'scheme' => env('PUSHER_SCHEME', 'http'),
                'encrypted' => false,
                'useTLS' => false,
                'curl_options' => [
                    CURLOPT_SSL_VERIFYPEER => false,
                ],
            ],
        ],

        'log' => [
            'driver' => 'log',
        ],

        'null' => [
            'driver' => 'null',
        ],
    ],
];

Configuración Soketi (soketi-config.json)
{
  "debug": false,
  "port": 6001,
  "host": "0.0.0.0",
  "metrics": {
    "enabled": true,
    "port": 9601
  },
  "app_manager": {
    "driver": "array",
    "array": [
      {
        "id": "local-app",
        "key": "local-key",
        "secret": "local-secret",
        "max_connections": 100,
        "enable_client_messages": false,
        "enabled": true
      }
    ]
  },
  "cors": {
    "credentials": true,
    "origin": ["http://localhost:5173"],
    "methods": ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
  }
}

> Regla de consistencia: Los valores de id, key y secret deben ser idénticos a los definidos en las variables PUSHER_* del archivo .env.
> 
Canales de Difusión (routes/channels.php)
<?php

use Illuminate\Support\Facades\Broadcast;

// Actualizaciones de tarjetas y estados de feedback
Broadcast::channel('feedback-updates', function () {
    return true; // Acceso para usuarios autenticados
});

// Sincronización del directorio de integrantes
Broadcast::channel('team-updates', function () {
    return true;
});

Habilita BroadcastServiceProvider en bootstrap/providers.php:
<?php

return [
    App\Providers\AppServiceProvider::class,
    App\Providers\BroadcastServiceProvider::class,
];

Autenticación y Control de Acceso
Configuración Sanctum (config/sanctum.php)
<?php

return [
    'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', 'localhost,localhost:5173,127.0.0.1')),
    'guard' => ['web'],
    'expiration' => null,
    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies' => Illuminate\Cookie\Middleware\EncryptCookies::class,
        'validate_csrf_token' => Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
    ],
];

Middleware IsAdmin (app/Http/Middleware/IsAdmin.php)
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class IsAdmin
{
    public function handle(Request $request, Closure $next): Response
    {
        if (!$request->user() || $request->user()->role !== 'admin') {
            return response()->json([
                'message' => 'Acceso denegado. Se requieren permisos de administrador.'
            ], 403);
        }

        return $next($request);
    }
}

Registro del alias en bootstrap/app.php:
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'isAdmin' => \App\Http\Middleware\IsAdmin::class,
    ]);
})

Procesamiento Asíncrono (config/queue.php)
Para evitar bloquear las respuestas HTTP al despachar eventos a Soketi, las tareas se procesan en segundo plano:
<?php

return [
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

    'failed' => [
        'driver' => env('QUEUE_FAILED_DRIVER', 'database-uuids'),
        'database' => env('DB_CONNECTION', 'mysql'),
        'table' => 'failed_jobs',
    ],
];

Script de Inicialización (backend/start.sh)
#!/bin/bash
set -e

echo "🚀 Iniciando servicios de RetroApp Backend..."

# Dependencias Composer
composer install --no-dev --optimize-autoloader --no-interaction

# Generar APP_KEY si no existe
if [ ! -f .env ]; then
    cp .env.example .env
fi
php artisan key:generate --force

# Caché de configuración y rutas
php artisan config:cache
php artisan route:cache

# Esperar disponibilidad de base de datos
until mysqladmin ping -h"$DB_HOST" -u"$DB_USERNAME" -p"$DB_PASSWORD" --silent; do
    echo "⏳ Esperando conexión con MySQL..."
    sleep 2
done

# Migraciones y datos iniciales
php artisan migrate --force
php artisan db:seed --force

# Iniciar Queue Worker en segundo plano
php artisan queue:work --daemon &
QUEUE_PID=$!

# Iniciar servidor WebSocket (Soketi)
npx soketi start --config=/var/www/html/soketi-config.json --port=6001 &
SOKETI_PID=$!

# Iniciar servidor API Laravel
php artisan serve --host=0.0.0.0 --port=8000 &
LARAVEL_PID=$!

# Control de señales para apagado limpio
cleanup() {
    echo "🛑 Deteniendo servicios..."
    kill $QUEUE_PID $SOKETI_PID $LARAVEL_PID 2>/dev/null || true
    exit 0
}
trap cleanup SIGINT SIGTERM

echo "✅ RetroApp Backend listo en http://localhost:8000"
echo "📡 Soketi activo en ws://localhost:6001"

wait

Diagnóstico y Resolución de Problemas
| Incidencia | Causa Habitual | Validación y Solución |
|---|---|---|
| Eventos no llegan al frontend | Soketi detenido o worker inactivo | Comprobar puerto con curl http://localhost:6001/ y verificar proceso con docker compose exec backend ps aux | grep queue. |
| Error 401 Unauthorized / 419 CSRF | Dominios stateful o CORS desalineados | Asegurar que SANCTUM_STATEFUL_DOMAINS contenga el host exacto del frontend (localhost:5173). |
| Error de conexión a BD | MySQL no ha completado el inicio | Confirmar estado del contenedor con docker compose ps database y revisar credenciales en .env. |
| Error de permisos en logs/cache | Permisos insuficientes en /storage | Ejecutar chmod -R 775 storage bootstrap/cache && chown -R www-data:www-data storage bootstrap/cache. |
