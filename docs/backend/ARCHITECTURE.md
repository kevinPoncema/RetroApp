# 🏗️ Backend Architecture - RetroApp

## Descripción General

El backend de **RetroApp** implementa una arquitectura modular por capas en Laravel 11 basada en principios SOLID, separación de responsabilidades y orientación a eventos (Event-Driven Architecture) para soportar reactividad en tiempo real.

---

## 🏛️ Flujo Arquitectónico

### 1. Ciclo de Vida HTTP (Request / Response)
```
Request ──→ Middleware (Auth/Role) ──→ Controller ──→ Service ──→ Repository ──→ Eloquent Model ──→ MySQL
                                           │
Response ◄── JSON Formatted ◄──────────────┘
```

### 2. Flujo de Eventos y Difusión (Broadcasting)
```
Mutación (Service) ──→ Event (ShouldBroadcast) ──→ Queue (DB Worker) ──→ Soketi (WebSocket) ──→ Frontend Client
```

---

## 📁 Estructura del Código Fuente

```
backend/src/
├── app/
│   ├── Events/              # Eventos transmitidos vía WebSocket
│   ├── Http/
│   │   ├── Controllers/Api/ # Endpoints y transformación de respuestas
│   │   ├── Middleware/      # Reglas de inspección (ej. IsAdmin)
│   │   ├── Requests/        # Validación y sanitización de entradas
│   │   └── Services/        # Reglas de negocio y orquestación
│   ├── Models/              # Entidades Eloquent y relaciones
│   └── Repositories/        # Abstracción de consultas y persistencia
├── config/                  # Ajustes de broadcasting, colas y auth
├── database/
│   ├── migrations/          # Esquemas relacionales
│   ├── seeders/             # Semillas de datos
│   └── factories/           # Generadores para testing
└── routes/
    ├── api.php              # Rutas REST
    └── channels.php         # Autorización de canales WebSocket
```

---

## 🧩 Capas y Componentes Core

### 1. Modelos de Dominio (`app/Models/`)

Representan las entidades principales y definen integridad relacional.

- **`User.php`**: Gestión de credenciales, roles (`admin`, `guest`) y tokens Sanctum.
- **`TeamMember.php`**: Perfiles de los integrantes del equipo de desarrollo sujetos a evaluación.
- **`Feedback.php`**: Notas clasificadas en `achievements`, `qualities` y `potential`.

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Factories\HasFactory;

class Feedback extends Model
{
    use HasFactory;

    protected $table = 'feedback';

    protected $fillable = [
        'target_id', 
        'owner_id', 
        'category', 
        'title', 
        'text'
    ];

    public function target(): BelongsTo
    {
        return $this->belongsTo(TeamMember::class, 'target_id');
    }

    public function owner(): BelongsTo
    {
        return $this->belongsTo(User::class, 'owner_id');
    }

    public function scopeByCategory($query, string $category)
    {
        return $query->where('category', $category);
    }
}
```

---

### 2. Capa de Repositorios (`app/Repositories/`)

Aísla las consultas directas al ORM de la lógica de negocio.

```php
namespace App\Repositories;

use App\Models\Feedback;
use Illuminate\Database\Eloquent\Collection;

class FeedbackRepo
{
    public function getAll(): Collection
    {
        return Feedback::with(['target', 'owner'])->get();
    }

    public function findById(int|string $id): Feedback
    {
        return Feedback::with(['target', 'owner'])->findOrFail($id);
    }

    public function create(array $data): Feedback
    {
        return Feedback::create($data);
    }

    public function update(int|string $id, array $data): Feedback
    {
        $feedback = Feedback::findOrFail($id);
        $feedback->update($data);
        return $feedback->load(['target', 'owner']);
    }

    public function delete(int|string $id): bool
    {
        return Feedback::findOrFail($id)->delete();
    }
}
```

---

### 3. Capa de Servicios (`app/Http/Services/`)

Centraliza validaciones de dominio, control de autoría y despacho de eventos.

```php
namespace App\Http\Services;

use App\Repositories\FeedbackRepo;
use App\Events\FeedbackCreated;
use App\Events\FeedbackUpdated;
use App\Events\FeedbackDeleted;
use App\Models\Feedback;
use Illuminate\Auth\Access\AuthorizationException;

class FeedbackServices
{
    public function __construct(
        protected FeedbackRepo $feedbackRepo
    ) {}

    public function createFeedback(array $data): Feedback
    {
        $data['owner_id'] = auth()->id();
        $feedback = $this->feedbackRepo->create($data);

        event(new FeedbackCreated($feedback));

        return $feedback;
    }

    public function updateFeedback(string $id, array $data): Feedback
    {
        $feedback = $this->feedbackRepo->findById($id);
        $this->authorizeOwnerOrAdmin($feedback);

        $updated = $this->feedbackRepo->update($id, $data);
        event(new FeedbackUpdated($updated));

        return $updated;
    }

    public function deleteFeedback(string $id): void
    {
        $feedback = $this->feedbackRepo->findById($id);
        $this->authorizeOwnerOrAdmin($feedback);

        $this->feedbackRepo->delete($id);
        event(new FeedbackDeleted($id));
    }

    private function authorizeOwnerOrAdmin(Feedback $feedback): void
    {
        $user = auth()->user();
        if ($feedback->owner_id !== $user->id && $user->role !== 'admin') {
            throw new AuthorizationException('No autorizado para modificar esta nota.');
        }
    }
}
```

---

### 4. Capa de Controladores (`app/Http/Controllers/Api/`)

Recepción HTTP sin lógica de negocio incrustada; delega directamente a Servicios y Form Requests.

```php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Services\FeedbackServices;
use App\Http\Requests\StoreFeedbackRequest;
use App\Http\Requests\UpdateFeedbackRequest;
use Illuminate\Http\JsonResponse;

class FeedbackController extends Controller
{
    public function __construct(
        protected FeedbackServices $feedbackServices
    ) {}

    public function store(StoreFeedbackRequest $request): JsonResponse
    {
        $feedback = $this->feedbackServices->createFeedback($request->validated());
        return response()->json($feedback, 201);
    }

    public function update(UpdateFeedbackRequest $request, string $id): JsonResponse
    {
        $feedback = $this->feedbackServices->updateFeedback($id, $request->validated());
        return response()->json($feedback, 200);
    }

    public function destroy(string $id): JsonResponse
    {
        $this->feedbackServices->deleteFeedback($id);
        return response()->json(['message' => 'Feedback eliminado con éxito.'], 200);
    }
}
```

---

### 5. Eventos de Tiempo Real (`app/Events/`)

Estructuran los paquetes difundidos vía Soketi hacia el frontend.

```php
namespace App\Events;

use App\Models\Feedback;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class FeedbackCreated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(public Feedback $feedback)
    {
        $this->feedback->load(['target', 'owner']);
    }

    public function broadcastOn(): array
    {
        return [new Channel('feedback-updates')];
    }

    public function broadcastAs(): string
    {
        return 'feedback.created';
    }

    public function broadcastWith(): array
    {
        return ['feedback' => $this->feedback];
    }
}
```

---

## 🗄️ Esquema de Base de Datos y Factories

### Migración de Feedback (`database/migrations/xxxx_create_feedback_table.php`)

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('feedback', function (Blueprint $table) {
            $table->id();
            $table->foreignId('target_id')->constrained('team_members')->cascadeOnDelete();
            $table->foreignId('owner_id')->nullable()->constrained('users')->cascadeOnDelete();
            $table->enum('category', ['achievements', 'qualities', 'potential']);
            $table->string('title', 50);
            $table->string('text', 300);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('feedback');
    }
};
```

### Factory de Pruebas (`database/factories/FeedbackFactory.php`)

```php
namespace Database\Factories;

use App\Models\Feedback;
use App\Models\TeamMember;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class FeedbackFactory extends Factory
{
    protected $model = Feedback::class;

    public function definition(): array
    {
        return [
            'target_id' => TeamMember::factory(),
            'owner_id'  => User::factory(),
            'category'  => fake()->randomElement(['achievements', 'qualities', 'potential']),
            'title'     => fake()->sentence(3),
            'text'      => fake()->paragraph(2),
        ];
    }
}
```
