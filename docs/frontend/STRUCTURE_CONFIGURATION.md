# 🎨 Frontend Structure & Configuration - 

## Descripción General

El frontend de RetroApp es una aplicación React moderna construida con Vite, que proporciona una interfaz de usuario interactiva para la gestión de retroalimentación de equipos en tiempo real. Utiliza Tailwind CSS para el diseño y Laravel Echo con Pusher.js para comunicación WebSocket.

## Estructura de Directorios

```
frontend/
├── public/                       # Archivos estáticos públicos
│   ├── vite.svg                 # Logo de Vite
│   └── favicon.ico              # Favicon de la aplicación
├── src/                         # Código fuente principal
│   ├── api/                     # Servicios de API y WebSocket
│   │   ├── apiService.js        # Cliente HTTP principal
│   │   ├── constants.js         # Constantes de API
│   │   ├── echo.js              # Configuración Laravel Echo
│   │   └── useAPI.js            # Hook personalizado para API
│   ├── assets/                  # Recursos estáticos
│   │   └── react.svg            # Logo de React
│   ├── components/              # Componentes React
│   │   ├── AuthenticationModal.jsx
│   │   ├── ErrorDisplay.jsx
│   │   ├── FeedbackFormModal.jsx
│   │   ├── FloatingButtons.jsx
│   │   ├── Header.jsx
│   │   ├── Loading.jsx
│   │   ├── LoginModal.jsx
│   │   ├── MainBoard.jsx
│   │   ├── MemberManagementModal.jsx
│   │   ├── MemberRow.jsx
│   │   ├── PostItDetailModal.jsx
│   │   ├── PostItNote.jsx
│   │   ├── UserInfo.jsx
│   │   └── index.js             # Barrel exports
│   ├── App.jsx                  # Componente raíz
│   ├── App.css                  # Estilos específicos de App
│   ├── index.css                # Estilos globales y Tailwind
│   └── main.jsx                 # Punto de entrada de la aplicación
├── package.json                 # Dependencias y scripts
├── vite.config.js              # Configuración de Vite
├── eslint.config.js            # Configuración de ESLint
├── index.html                  # Template HTML
└── dockerfile                  # Configuración Docker
```

## Configuración del Proyecto

### package.json

El proyecto utiliza las siguientes dependencias principales:

```json
{
  "name": "app",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",                    // Servidor de desarrollo
    "build": "vite build",            // Build de producción
    "lint": "eslint .",               // Linting del código
    "preview": "vite preview"         // Preview del build
  },
  "dependencies": {
    "laravel-echo": "^1.15.3",       // Cliente WebSocket para Laravel
    "lucide-react": "^0.554.0",      // Iconos SVG (actualmente no usado)
    "pusher-js": "^8.4.0-rc2",       // Cliente Pusher/Soketi
    "react": "^19.2.0",              // Librería React
    "react-dom": "^19.2.0"           // React DOM
  },
  "devDependencies": {
    "@eslint/js": "^9.39.1",         // ESLint core
    "@types/react": "^19.2.5",       // Types de React
    "@types/react-dom": "^19.2.3",   // Types de React DOM
    "@vitejs/plugin-react": "^5.1.1", // Plugin React para Vite
    "eslint": "^9.39.1",             // Linter
    "eslint-plugin-react-hooks": "^7.0.1", // Rules para React Hooks
    "eslint-plugin-react-refresh": "^0.4.24", // Rules para React Refresh
    "globals": "^16.5.0",            // Variables globales para ESLint
    "vite": "^7.2.4"                 // Build tool y dev server
  }
}
```

### Vite Configuration (vite.config.js)

Configuración básica de Vite para React:

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  
  // Configuración del servidor de desarrollo
  server: {
    port: 5173,                     // Puerto del frontend
    host: true,                     // Escuchar en todas las interfaces
    strictPort: true,               // Fallar si el puerto está ocupado
    
    // Configuración de proxy para API en desarrollo (opcional)
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        secure: false
      }
    }
  },

  // Configuración de build
  build: {
    outDir: 'dist',                 // Directorio de salida
    sourcemap: true,                // Generar sourcemaps
    
    // Optimización de chunks
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'websocket': ['laravel-echo', 'pusher-js']
        }
      }
    }
  },

  // Variables de entorno
  define: {
    'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV)
  }
})
```

### ESLint Configuration (eslint.config.js)

Configuración de linting para mantener calidad del código:

```javascript
import js from '@eslint/js'
import globals from 'globals'
import react from 'eslint-plugin-react'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'

export default [
  { ignores: ['dist'] },
  {
    files: ['**/*.{js,jsx}'],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
      parserOptions: {
        ecmaVersion: 'latest',
        ecmaFeatures: { jsx: true },
        sourceType: 'module',
      },
    },
    settings: { react: { version: '18.3' } },
    plugins: {
      react,
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...js.configs.recommended.rules,
      ...react.configs.recommended.rules,
      ...react.configs['jsx-runtime'].rules,
      ...reactHooks.configs.recommended.rules,
      'react/jsx-no-target-blank': 'off',
      'react-refresh/only-export-components': [
        'warn',
        { allowConstantExport: true },
      ],
    },
  },
]
```

### HTML Template (index.html)

Template base de la aplicación:

```html
<!doctype html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>FUNA FENIX - Retroalimentación de Equipo</title>
    
    <!-- Meta tags adicionales -->
    <meta name="description" content="Plataforma de retroalimentación para el equipo FENIX" />
    <meta name="theme-color" content="#000000" />
    
    <!-- Preconnect para APIs externas -->
    <link rel="preconnect" href="http://localhost:8000" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

### Entry Point (main.jsx)

Punto de entrada de la aplicación React:

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'
import './index.css'

// Crear root de React 18
const root = ReactDOM.createRoot(document.getElementById('root'))

// Renderizar aplicación
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)

// Hot Module Replacement para desarrollo
if (import.meta.hot) {
  import.meta.hot.accept()
}
```

## Estilos y CSS

### Global Styles (index.css)

Configuración de Tailwind CSS y estilos globales:

```css
/* Importar Tailwind CSS */
@import 'tailwindcss/base';
@import 'tailwindcss/components'; 
@import 'tailwindcss/utilities';

/* Variables CSS personalizadas */
:root {
  /* Colores del tema */
  --color-primary: #3b82f6;
  --color-secondary: #10b981;
  --color-danger: #ef4444;
  --color-warning: #f59e0b;
  
  /* Gradientes */
  --gradient-primary: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  --gradient-secondary: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
  
  /* Sombras */
  --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  
  /* Fuentes */
  --font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

/* Reset y configuración base */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: var(--font-family);
  line-height: 1.6;
  color: #333;
  background: var(--gradient-primary);
  min-height: 100vh;
}

/* Clases utilitarias personalizadas */
.line-clamp-1 {
  display: -webkit-box;
  -webkit-line-clamp: 1;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.line-clamp-2 {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.line-clamp-3 {
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.line-clamp-4 {
  display: -webkit-box;
  -webkit-line-clamp: 4;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* Animaciones personalizadas */
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes slideIn {
  from { transform: translateX(-100%); }
  to { transform: translateX(0); }
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}

.animate-fadeIn {
  animation: fadeIn 0.3s ease-out;
}

.animate-slideIn {
  animation: slideIn 0.3s ease-out;
}

/* Scrollbar personalizado */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: rgba(0, 0, 0, 0.1);
  border-radius: 4px;
}

::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}

/* Estilos para elementos específicos de la app */
.board-container {
  max-width: 1400px;
  margin: 0 auto;
  padding: 2rem 1rem;
}

.post-it-note {
  background: linear-gradient(145deg, #ffeaa7, #fdcb6e);
  box-shadow: 
    0 4px 8px rgba(0, 0, 0, 0.1),
    0 6px 20px rgba(0, 0, 0, 0.15);
  transform: rotate(-1deg);
  transition: all 0.3s ease;
}

.post-it-note:hover {
  transform: rotate(0deg) scale(1.05);
  box-shadow: 
    0 8px 16px rgba(0, 0, 0, 0.15),
    0 12px 40px rgba(0, 0, 0, 0.2);
}

.member-avatar {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border: 3px solid rgba(255, 255, 255, 0.2);
}

/* Responsive breakpoints */
@media (max-width: 768px) {
  .board-container {
    padding: 1rem 0.5rem;
  }
  
  .post-it-note {
    transform: rotate(0deg);
  }
}

/* Estados de carga */
.loading-skeleton {
  background: linear-gradient(90deg, #f0f0f0 25%, transparent 37%, #f0f0f0 63%);
  background-size: 400% 100%;
  animation: shimmer 1.4s ease infinite;
}

@keyframes shimmer {
  0% { background-position: 100% 0; }
  100% { background-position: -100% 0; }
}

/* Estados de error */
.error-border {
  border: 2px solid var(--color-danger);
  box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.1);
}

/* Focus states para accesibilidad */
.focus-visible {
  outline: 2px solid var(--color-primary);
  outline-offset: 2px;
}

/* Print styles */
@media print {
  .no-print {
    display: none !important;
  }
  
  body {
    background: white;
    color: black;
  }
}
```

### Component Styles (App.css)

Estilos específicos para el componente App:

```css
/* Estilos del componente App */
.app-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  position: relative;
  overflow-x: hidden;
}

/* Header styles */
.app-header {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
  padding: 1rem 2rem;
  position: sticky;
  top: 0;
  z-index: 30;
}

.app-header h1 {
  background: linear-gradient(45deg, #fff, #f8f9fa);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  font-size: 2.5rem;
  font-weight: 800;
  text-align: center;
}

/* Floating buttons container */
.floating-buttons {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  z-index: 20;
}

.floating-button {
  background: var(--color-primary);
  color: white;
  border: none;
  border-radius: 50%;
  width: 60px;
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: var(--shadow-lg);
  cursor: pointer;
  transition: all 0.3s ease;
}

.floating-button:hover {
  transform: scale(1.1);
  background: #2563eb;
}

.floating-button:active {
  transform: scale(0.95);
}

/* Modal overlay */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(5px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 50;
  animation: fadeIn 0.2s ease-out;
}

/* Modal content */
.modal-content {
  background: white;
  border-radius: 1rem;
  padding: 2rem;
  max-width: 90vw;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: var(--shadow-lg);
  animation: slideIn 0.3s ease-out;
}

/* Responsive adjustments */
@media (max-width: 768px) {
  .app-header {
    padding: 1rem;
  }
  
  .app-header h1 {
    font-size: 1.8rem;
  }
  
  .floating-buttons {
    bottom: 1rem;
    right: 1rem;
  }
  
  .floating-button {
    width: 50px;
    height: 50px;
  }
  
  .modal-content {
    margin: 1rem;
    padding: 1.5rem;
  }
}

/* Dark mode support (futuro) */
@media (prefers-color-scheme: dark) {
  :root {
    --color-text: #f8f9fa;
    --color-background: #1a1a1a;
  }
}

/* Reduced motion para accesibilidad */
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}

/* High contrast mode */
@media (prefers-contrast: high) {
  .post-it-note {
    border: 2px solid black;
  }
  
  .floating-button {
    border: 2px solid white;
  }
}
```

## Docker Configuration

### Dockerfile

Configuración para containerización del frontend:

```dockerfile
# Usar imagen oficial de Node.js
FROM node:18-alpine

# Establecer directorio de trabajo
WORKDIR /app

# Copiar archivos de dependencias
COPY package*.json ./

# Instalar dependencias
RUN npm ci --only=production

# Copiar código fuente
COPY . .

# Exponer puerto
EXPOSE 5173

# Comando para desarrollo
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]

# Para build de producción:
# FROM node:18-alpine as build
# WORKDIR /app
# COPY package*.json ./
# RUN npm ci
# COPY . .
# RUN npm run build

# FROM nginx:alpine
# COPY --from=build /app/dist /usr/share/nginx/html
# EXPOSE 80
# CMD ["nginx", "-g", "daemon off;"]
```

## Environment Variables

Aunque Vite no utiliza un archivo `.env` tradicional, las variables de entorno pueden configurarse:

```bash
# Variables de desarrollo (.env.development)
VITE_API_BASE_URL=http://localhost:8000
VITE_WS_HOST=localhost
VITE_WS_PORT=6001
VITE_WS_KEY=local-key
VITE_APP_NAME=FUNA FENIX

# Variables de producción (.env.production)
VITE_API_BASE_URL=https://api.funafenix.com
VITE_WS_HOST=ws.funafenix.com
VITE_WS_PORT=443
VITE_WS_KEY=production-key
VITE_APP_NAME=FUNA FENIX
```

**Nota**: Las variables deben tener el prefijo `VITE_` para ser accesibles en el cliente.

## Build y Deploy

### Scripts de NPM

```bash
# Desarrollo
npm run dev          # Inicia servidor de desarrollo en puerto 5173

# Build
npm run build        # Genera build optimizado en carpeta 'dist'

# Preview
npm run preview      # Preview del build en servidor local

# Linting
npm run lint         # Ejecuta ESLint para revisar código
```

### Optimizaciones de Build

El build de producción incluye:
- **Minificación**: CSS y JS minificados
- **Tree Shaking**: Eliminación de código no utilizado
- **Code Splitting**: División automática de chunks
- **Asset Optimization**: Optimización de imágenes y recursos
- **Gzip Compression**: Compresión de archivos estáticos

## Performance Considerations

### 1. Lazy Loading
Carga diferida de componentes pesados:

```jsx
const HeavyComponent = lazy(() => import('./HeavyComponent'))
```

### 2. Memoización
Uso de React.memo para componentes que no cambian frecuentemente:

```jsx
const PostItNote = React.memo(({ feedback, ...props }) => {
  // Component implementation
})
```

### 3. Bundle Analysis
Para analizar el tamaño del bundle:

```bash
npm install --save-dev rollup-plugin-visualizer
npm run build -- --analyze
```

### 4. WebSocket Optimization
Gestión eficiente de conexiones WebSocket con reconexión automática y limpieza adecuada.
