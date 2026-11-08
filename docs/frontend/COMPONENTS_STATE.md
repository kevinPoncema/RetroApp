# 🧩 Componentes y Gestión de Estado - RetroApp Frontend

## Descripción General

El frontend de **RetroApp** implementa una arquitectura modular basada en React 18 orientada a la sincronización en tiempo real. Utiliza elevación de estado (*lifting state up*), composición de componentes limpios y comunicación reactiva con WebSockets para las dinámicas de retrospectiva del equipo de desarrollo.

---

## 🏛️ Jerarquía de Componentes

```
App (Estado Global y Coordinación WebSocket)
├── Header (Identidad y estado de conexión en vivo)
├── UserInfo (Sesión activa, rol y logout)
├── FloatingButtons (Acciones de feedback y administración)
├── MainBoard (Tablero principal interactivo)
│   └── MemberRow[] (Fila por cada integrante del equipo)
│       └── PostItNote[] (Tarjetas de feedback por categoría)
├── AuthenticationModal (Login admin / invitado)
│   ├── LoginModal
│   └── ErrorDisplay
├── FeedbackFormModal (Creación de notas)
├── MemberManagementModal (CRUD de integrantes para admin)
├── PostItDetailModal (Vista expandida de nota)
└── Loading (Feedback visual de operaciones asíncronas)
```

---

## 🔄 Flujo de Datos y Sincronización

```
[WebSocket: Soketi] ──(Eventos en vivo)──┐
                                         ▼
[Acción de Usuario] ──→ useAPI Hook ──→ App State (Feedbacks / Members)
                             │                 │
                             ▼                 ▼
                       apiService (HTTP)    MainBoard ──→ MemberRow ──→ PostItNote
```

---

## 🧩 Componentes Core

### 1. Tablero Principal (`src/components/MainBoard.jsx`)

Renderiza las filas de los integrantes del equipo o un estado vacío si no hay registros.

```jsx
import React from 'react';
import { Users } from 'lucide-react';
import MemberRow from './MemberRow.jsx';

const MainBoard = ({ 
    teamMembers, 
    feedbackData, 
    userId, 
    deleteFeedback, 
    onOpenDetail, 
    webSocketFeedbackIds 
}) => {
    if (!teamMembers.length) {
        return (
            <div className="text-center p-10 text-white/70">
                <Users className="mx-auto mb-4 text-white/50" size={48} />
                <p className="text-xl font-semibold">No hay miembros registrados en el equipo de desarrollo.</p>
                <p className="text-base mt-2">Utiliza la sección de administración para agregar colaboradores.</p>
            </div>
        );
    }

    return (
        <main className="w-full space-y-4">
            {teamMembers.map((member) => (
                <MemberRow
                    key={member.id}
                    member={member}
                    feedbackData={feedbackData}
                    currentUserId={userId}
                    deleteFeedback={deleteFeedback}
                    onOpenDetail={onOpenDetail}
                    webSocketFeedbackIds={webSocketFeedbackIds}
                />
            ))}
        </main>
    );
};

export default React.memo(MainBoard);
```

---

### 2. Fila de Miembro (`src/components/MemberRow.jsx`)

Agrupa y categoriza las notas asociadas a cada integrante usando `useMemo` para evitar reclasificaciones innecesarias en cada re-renderizado.

```jsx
import React, { useMemo } from 'react';
import { ThumbsUp, User, TrendingUp } from 'lucide-react';
import PostItNote from './PostItNote.jsx';

const CATEGORIES = [
    { id: 'achievements', title: 'Logros', icon: ThumbsUp, color: 'bg-emerald-200' },
    { id: 'qualities', title: 'Cualidades', icon: User, color: 'bg-sky-200' },
    { id: 'potential', title: 'Crecimiento', icon: TrendingUp, color: 'bg-violet-200' }
];

const MemberRow = ({ 
    member, 
    feedbackData, 
    currentUserId, 
    deleteFeedback, 
    onOpenDetail, 
    webSocketFeedbackIds 
}) => {
    const feedbackByCategory = useMemo(() => {
        const groups = { achievements: [], qualities: [], potential: [] };
        feedbackData
            .filter((item) => item.target_id === member.id)
            .forEach((item) => {
                if (groups[item.category]) {
                    groups[item.category].push(item);
                }
            });
        return groups;
    }, [feedbackData, member.id]);

    return (
        <div className="flex flex-col lg:flex-row border-b border-white/10 py-4 gap-4 hover:bg-white/5 transition-colors">
            {/* Información del integrante */}
            <div className="w-full lg:w-48 flex lg:flex-col items-center gap-3 shrink-0">
                <div className="w-14 h-14 rounded-full bg-slate-700 text-white flex items-center justify-center text-xl font-bold">
                    {member.name.charAt(0).toUpperCase()}
                </div>
                <div>
                    <h3 className="font-bold text-white text-base">{member.name}</h3>
                    <p className="text-xs text-slate-400">{member.role}</p>
                </div>
            </div>

            {/* Columnas de categorías */}
            <div className="flex-1 grid grid-cols-1 md:grid-cols-3 gap-3">
                {CATEGORIES.map((cat) => (
                    <div key={cat.id} className="border-l border-white/10 pl-3">
                        <div className="flex items-center gap-2 mb-2 text-xs font-semibold uppercase tracking-wider text-slate-300">
                            <cat.icon size={14} />
                            <span>{cat.title}</span>
                        </div>
                        <div className="flex flex-wrap gap-2">
                            {feedbackByCategory[cat.id]?.length > 0 ? (
                                feedbackByCategory[cat.id].map((fb) => (
                                    <PostItNote
                                        key={fb.id}
                                        feedback={fb}
                                        colorClass={cat.color}
                                        isAuthor={fb.owner_id === currentUserId}
                                        onDelete={deleteFeedback}
                                        onOpenDetail={onOpenDetail}
                                        webSocketFeedbackIds={webSocketFeedbackIds}
                                    />
                                ))
                            ) : (
                                <span className="text-xs text-white/30 italic">Sin registros...</span>
                            )}
                        </div>
                    </div>
                ))}
            </div>
        </div>
    );
};

export default React.memo(MemberRow);
```

---

### 3. Tarjeta de Feedback (`src/components/PostItNote.jsx`)

Maneja la representación visual tipo *post-it*, el cálculo de rotación estética y el control de borrado basado en permisos (autor o rol administrador).

```jsx
import React, { useMemo, useState } from 'react';

const PostItNote = ({ 
    feedback, 
    colorClass, 
    onDelete, 
    isAuthor, 
    onOpenDetail, 
    webSocketFeedbackIds 
}) => {
    const [isDeleting, setIsDeleting] = useState(false);
    const currentUser = useMemo(() => {
        try {
            return JSON.parse(localStorage.getItem('user'));
        } catch {
            return null;
        }
    }, []);

    const canDelete = isAuthor || currentUser?.role === 'admin';
    const isNew = webSocketFeedbackIds?.new?.has(feedback.id);
    const isPendingDelete = webSocketFeedbackIds?.deleting?.has(feedback.id);

    const rotation = useMemo(() => {
        return (Math.random() * 2 - 1).toFixed(2);
    }, []);

    const handleDelete = (e) => {
        e.stopPropagation();
        setIsDeleting(true);
        setTimeout(() => onDelete(feedback.id), 250);
    };

    return (
        <div
            onClick={() => onOpenDetail(feedback)}
            style={{ transform: `rotate(${rotation}deg)` }}
            className={`relative p-3 w-40 h-44 shadow-lg rounded text-slate-800 flex flex-col justify-between cursor-pointer transition-all duration-200 hover:scale-105 ${colorClass} ${
                isNew ? 'ring-2 ring-emerald-500 animate-pulse' : ''
            } ${isDeleting || isPendingDelete ? 'opacity-20 scale-90' : ''}`}
        >
            {canDelete && (
                <button
                    onClick={handleDelete}
                    className="absolute top-1 right-1 w-5 h-5 rounded-full bg-black/20 hover:bg-rose-600 hover:text-white flex items-center justify-center text-xs transition-colors"
                    title="Eliminar"
                >
                    ✕
                </button>
            )}

            <div>
                <h5 className="font-bold text-sm leading-tight line-clamp-2 mb-1">
                    {feedback.title}
                </h5>
                <p className="text-xs leading-relaxed line-clamp-4 text-slate-700">
                    {feedback.text}
                </p>
            </div>

            <div className="flex justify-between items-center text-[10px] text-slate-600 border-t border-black/10 pt-1 mt-auto">
                <span className="truncate">{feedback.owner?.name || 'Anónimo'}</span>
                <span>{new Date(feedback.created_at).toLocaleDateString()}</span>
            </div>
        </div>
    );
};

export default React.memo(PostItNote);
```

---

## ⚡ Patrones de Optimización y Rendimiento

1. **Memoización con `React.memo`**: Evita renderizados en cascada en `MemberRow` y `PostItNote` cuando entran actualizaciones por WebSocket asociadas a otros miembros del equipo.
2. **Animaciones Efímeras con `Set`**: Se emplea `Set` en el estado global para rastrear IDs en animación activa (`new` o `deleting`), garantizando búsquedas en tiempo constante $O(1)$.
3. **Persistencia y Control de Sesión**: La recuperación de credenciales y tokens se ejecuta en el ciclo inicial para prevenir saltos de maquetación (*layout shifts*) en la interfaz.
