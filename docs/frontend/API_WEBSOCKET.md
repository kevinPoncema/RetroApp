# 🌐 API & WebSocket Client - RetroApp Frontend

## Descripción General

El cliente de red de **RetroApp** gestiona la comunicación HTTP con la API REST de Laravel 11 y la sincronización bidireccional en tiempo real con Soketi mediante Laravel Echo. Proporciona una interfaz unificada para interactuar con las notas de retroalimentación y los integrantes del equipo de desarrollo.

---

## 🏗️ Arquitectura de Comunicación

```
Componentes React (Tablero, Modales, Notas)
                      │
                      ▼
               Hook: useAPI()
                      │
         ┌────────────┴────────────┐
         ▼                         ▼
   apiService.js                echo.js
   (Cliente HTTP / Fetch)       (Laravel Echo / WebSocket)
         │                         │
         ▼                         ▼
   Laravel 11 REST API        Soketi (Puerto 6001)
         │                         │
         └────────────┬────────────┘
                      ▼
               MySQL Database
```

---

## 📡 Constantes del Sistema (`src/api/constants.js`)

Centraliza endpoints, canales de difusión, categorías del equipo de desarrollo y mensajes estándar.

```javascript
export const API_CONFIG = {
    BASE_URL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000/api',
    TIMEOUT: 15000,
};

export const WEBSOCKET_CONFIG = {
    HOST: import.meta.env.VITE_WS_HOST || 'localhost',
    PORT: Number(import.meta.env.VITE_WS_PORT) || 6001,
    APP_KEY: import.meta.env.VITE_WS_APP_KEY || 'local-key',
    FORCE_TLS: false,
};

export const CHANNELS = {
    FEEDBACK_UPDATES: 'feedback-updates',
    TEAM_UPDATES: 'team-updates',
};

export const EVENTS = {
    FEEDBACK_CREATED: '.feedback.created',
    FEEDBACK_UPDATED: '.feedback.updated',
    FEEDBACK_DELETED: '.feedback.deleted',
    TEAM_MEMBER_CREATED: '.team-member.created',
    TEAM_MEMBER_UPDATED: '.team-member.updated',
    TEAM_MEMBER_DELETED: '.team-member.deleted',
};

export const FEEDBACK_CATEGORIES = [
    {
        id: 'achievements',
        name: 'Logros',
        description: 'Hitos, entregables clave y metas alcanzadas',
        badgeColor: 'bg-emerald-100 text-emerald-800 border-emerald-300',
    },
    {
        id: 'qualities',
        name: 'Cualidades',
        description: 'Habilidades interpersonales y colaboración técnica',
        badgeColor: 'bg-sky-100 text-sky-800 border-sky-300',
    },
    {
        id: 'potential',
        name: 'Oportunidades de Mejora',
        description: 'Áreas de crecimiento profesional e impacto en el equipo',
        badgeColor: 'bg-amber-100 text-amber-800 border-amber-300',
    },
];
```

---

## 🔌 Cliente HTTP (`src/api/apiService.js`)

Implementación nativa sobre `fetch` con inyección automática de Bearer tokens, detección de respuestas 401 y parseo seguro de errores de validación de Laravel.

```javascript
import { API_CONFIG } from './constants.js';

class APIService {
    constructor() {
        this.token = localStorage.getItem('authToken');
        this.user = JSON.parse(localStorage.getItem('user')) || null;
    }

    getHeaders() {
        const headers = {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
            'X-Requested-With': 'XMLHttpRequest',
        };

        if (this.token) {
            headers['Authorization'] = `Bearer ${this.token}`;
        }

        return headers;
    }

    async request(endpoint, options = {}) {
        const url = `${API_CONFIG.BASE_URL}${endpoint}`;
        const config = {
            headers: this.getHeaders(),
            ...options,
        };

        try {
            const response = await fetch(url, config);
            const contentType = response.headers.get('content-type');
            let data = null;

            if (contentType && contentType.includes('application/json')) {
                const text = await response.text();
                data = text.trim() ? JSON.parse(text) : null;
            }

            if (!response.ok) {
                if (response.status === 401) {
                    this.clearAuth();
                    window.dispatchEvent(new CustomEvent('auth:unauthorized'));
                    throw new Error('Sesión expirada. Por favor, autentícate nuevamente.');
                }

                if (response.status === 403) {
                    throw new Error(data?.message || 'No tienes permisos para realizar esta acción.');
                }

                if (response.status === 422 && data?.errors) {
                    const validationMessages = Object.values(data.errors).flat().join('. ');
                    throw new Error(validationMessages);
                }

                throw new Error(data?.message || `Error del servidor HTTP ${response.status}`);
            }

            return data;
        } catch (error) {
            if (error.name === 'TypeError' && error.message.includes('fetch')) {
                throw new Error('No fue posible contactar con la API del backend.');
            }
            throw error;
        }
    }

    // ===== AUTENTICACIÓN =====
    async loginAdmin(email, password) {
        const data = await this.request('/auth/login-admin', {
            method: 'POST',
            body: JSON.stringify({ email, password }),
        });
        this.setSession(data.token, data.user);
        return data;
    }

    async loginGuest(name = null, hash = null) {
        const data = await this.request('/auth/login-guest', {
            method: 'POST',
            body: JSON.stringify({ name, hash }),
        });
        this.setSession(data.token, data.user);
        return data;
    }

    async logout() {
        try {
            await this.request('/auth/logout', { method: 'POST' });
        } finally {
            this.clearAuth();
        }
    }

    async getCurrentUser() {
        const user = await this.request('/user');
        this.user = user;
        localStorage.setItem('user', JSON.stringify(user));
        return user;
    }

    setSession(token, user) {
        this.token = token;
        this.user = user;
        localStorage.setItem('authToken', token);
        localStorage.setItem('user', JSON.stringify(user));
    }

    clearAuth() {
        this.token = null;
        this.user = null;
        localStorage.removeItem('authToken');
        localStorage.removeItem('user');
    }

    // ===== GESTIÓN DEL EQUIPO DE DESARROLLO =====
    getTeamMembers() {
        return this.request('/team-members');
    }

    getTeamMembersWithFeedbacks() {
        return this.request('/team-members-with-feedbacks');
    }

    createTeamMember(payload) {
        return this.request('/team-members', {
            method: 'POST',
            body: JSON.stringify(payload),
        });
    }

    updateTeamMember(id, payload) {
        return this.request(`/team-members/${id}`, {
            method: 'PUT',
            body: JSON.stringify(payload),
        });
    }

    deleteTeamMember(id) {
        return this.request(`/team-members/${id}`, { method: 'DELETE' });
    }

    // ===== RETROALIMENTACIÓN / NOTAS =====
    getFeedbacks() {
        return this.request('/feedbacks');
    }

    createFeedback(payload) {
        return this.request('/feedbacks', {
            method: 'POST',
            body: JSON.stringify(payload),
        });
    }

    updateFeedback(id, payload) {
        return this.request(`/feedbacks/${id}`, {
            method: 'PUT',
            body: JSON.stringify(payload),
        });
    }

    deleteFeedback(id) {
        return this.request(`/feedbacks/${id}`, { method: 'DELETE' });
    }
}

export const apiService = new APIService();
export default apiService;
```

---

## ⚡ Cliente WebSocket con Laravel Echo (`src/api/echo.js`)

Instancia singleton configurada para interactuar con Soketi mediante protocolo Pusher.

```javascript
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';
import { WEBSOCKET_CONFIG } from './constants.js';

window.Pusher = Pusher;

export const echo = new Echo({
    broadcaster: 'pusher',
    key: WEBSOCKET_CONFIG.APP_KEY,
    wsHost: WEBSOCKET_CONFIG.HOST,
    wsPort: WEBSOCKET_CONFIG.PORT,
    wssPort: WEBSOCKET_CONFIG.PORT,
    forceTLS: WEBSOCKET_CONFIG.FORCE_TLS,
    encrypted: false,
    disableStats: true,
    enabledTransports: ['ws', 'wss'],
    cluster: 'mt1', // Requerido sintácticamente por el driver pusher
});

// Enlace de observadores de estado de red
echo.connector.pusher.connection.bind('connected', () => {
    window.dispatchEvent(new CustomEvent('websocket:connected'));
});

echo.connector.pusher.connection.bind('disconnected', () => {
    window.dispatchEvent(new CustomEvent('websocket:disconnected'));
});

echo.connector.pusher.connection.bind('error', (err) => {
    window.dispatchEvent(new CustomEvent('websocket:error', { detail: err }));
});

export const updateEchoAuthToken = (token) => {
    if (!token) return;
    echo.connector.pusher.config.auth = {
        headers: {
            Authorization: `Bearer ${token}`,
        },
    };
};

export default echo;
```

---

## 🪝 Hook React Unificado (`src/api/useAPI.js`)

Encapsula la carga de datos del tablero, mutaciones HTTP y suscripciones reactivas a canales WebSocket en un ciclo de vida desacoplado.

```javascript
import { useState, useEffect, useCallback } from 'react';
import { apiService } from './apiService.js';
import { echo, updateEchoAuthToken } from './echo.js';
import { CHANNELS, EVENTS } from './constants.js';

export const useAPI = () => {
    const [teamData, setTeamData] = useState([]);
    const [currentUser, setCurrentUser] = useState(() => apiService.user);
    const [connectionStatus, setConnectionStatus] = useState('connecting');
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState(null);

    // Carga inicial del tablero completo
    const fetchBoardData = useCallback(async () => {
        setIsLoading(true);
        setError(null);
        try {
            const data = await apiService.getTeamMembersWithFeedbacks();
            setTeamData(data);
        } catch (err) {
            setError(err.message);
        } finally {
            setIsLoading(false);
        }
    }, []);

    // Sincronización de eventos en tiempo real
    useEffect(() => {
        if (apiService.token) {
            updateEchoAuthToken(apiService.token);
        }

        const handleConnected = () => setConnectionStatus('connected');
        const handleDisconnected = () => setConnectionStatus('disconnected');

        window.addEventListener('websocket:connected', handleConnected);
        window.addEventListener('websocket:disconnected', handleDisconnected);

        // Canal de notas de retroalimentación
        const feedbackChannel = echo.channel(CHANNELS.FEEDBACK_UPDATES);

        feedbackChannel.listen(EVENTS.FEEDBACK_CREATED, ({ feedback }) => {
            setTeamData((prev) =>
                prev.map((member) => {
                    if (member.id === feedback.target_id) {
                        const exists = member.feedbacks?.some((f) => f.id === feedback.id);
                        if (exists) return member;
                        return {
                            ...member,
                            feedbacks: [feedback, ...(member.feedbacks || [])],
                        };
                    }
                    return member;
                })
            );
        });

        feedbackChannel.listen(EVENTS.FEEDBACK_UPDATED, ({ feedback }) => {
            setTeamData((prev) =>
                prev.map((member) => {
                    if (member.id === feedback.target_id) {
                        return {
                            ...member,
                            feedbacks: (member.feedbacks || []).map((f) =>
                                f.id === feedback.id ? feedback : f
                            ),
                        };
                    }
                    return member;
                })
            );
        });

        feedbackChannel.listen(EVENTS.FEEDBACK_DELETED, ({ id, feedback }) => {
            const targetId = id || feedback?.id;
            setTeamData((prev) =>
                prev.map((member) => ({
                    ...member,
                    feedbacks: (member.feedbacks || []).filter((f) => f.id !== Number(targetId)),
                }))
            );
        });

        // Canal del directorio del equipo de desarrollo
        const teamChannel = echo.channel(CHANNELS.TEAM_UPDATES);

        teamChannel.listen(EVENTS.TEAM_MEMBER_CREATED, ({ teamMember }) => {
            setTeamData((prev) => {
                const exists = prev.some((m) => m.id === teamMember.id);
                return exists ? prev : [...prev, { ...teamMember, feedbacks: [] }];
            });
        });

        teamChannel.listen(EVENTS.TEAM_MEMBER_UPDATED, ({ teamMember }) => {
            setTeamData((prev) =>
                prev.map((m) => (m.id === teamMember.id ? { ...m, ...teamMember } : m))
            );
        });

        teamChannel.listen(EVENTS.TEAM_MEMBER_DELETED, ({ id }) => {
            setTeamData((prev) => prev.filter((m) => m.id !== Number(id)));
        });

        return () => {
            window.removeEventListener('websocket:connected', handleConnected);
            window.removeEventListener('websocket:disconnected', handleDisconnected);
            echo.leave(CHANNELS.FEEDBACK_UPDATES);
            echo.leave(CHANNELS.TEAM_UPDATES);
        };
    }, []);

    // Operaciones expuestas hacia los componentes
    const authenticate = async (type = 'admin', credentials = {}) => {
        setIsLoading(true);
        setError(null);
        try {
            const response =
                type === 'admin'
                    ? await apiService.loginAdmin(credentials.email, credentials.password)
                    : await apiService.loginGuest(credentials.name, credentials.hash);

            setCurrentUser(response.user);
            updateEchoAuthToken(response.token);
            await fetchBoardData();
            return response;
        } catch (err) {
            setError(err.message);
            throw err;
        } finally {
            setIsLoading(false);
        }
    };

    const logout = async () => {
        try {
            await apiService.logout();
        } finally {
            setCurrentUser(null);
            setTeamData([]);
        }
    };

    const submitFeedback = (payload) => apiService.createFeedback(payload);
    const modifyFeedback = (id, payload) => apiService.updateFeedback(id, payload);
    const removeFeedback = (id) => apiService.deleteFeedback(id);

    return {
        teamData,
        currentUser,
        connectionStatus,
        isLoading,
        error,
        fetchBoardData,
        authenticate,
        logout,
        submitFeedback,
        modifyFeedback,
        removeFeedback,
    };
};
```
