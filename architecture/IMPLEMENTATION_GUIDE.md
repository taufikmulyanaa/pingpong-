# PingPong+ Implementation Guide

## Overview
Complete implementation guide for the PingPong+ table tennis social platform, covering all aspects from development setup to production deployment.

## 1. State Management Implementation

### Global State Architecture
```javascript
// src/Frontend/stores/AppStore.js
class AppStore extends Store {
  constructor() {
    super({
      // UI State
      currentPage: 'home',
      isLoading: false,
      isMobile: false,
      sidebarOpen: false,

      // User State
      currentUser: null,
      isAuthenticated: false,

      // Data State
      tournaments: [],
      clubs: [],
      venues: [],
      feedPosts: [],
      marketplaceItems: [],

      // UI State
      modals: {
        auth: { isOpen: false, mode: 'login' },
        tournament: { isOpen: false, data: null },
        club: { isOpen: false, data: null }
      },

      // Cache State
      cache: new Map(),
      lastUpdated: {}
    });

    this.initResponsiveHandling();
    this.initCacheManagement();
  }

  initResponsiveHandling() {
    const checkMobile = () => {
      const isMobile = window.innerWidth < 768;
      if (isMobile !== this.state.isMobile) {
        this.dispatch({ type: 'SET_MOBILE', payload: isMobile });
      }
    };

    window.addEventListener('resize', debounce(checkMobile, 250));
    checkMobile();
  }

  initCacheManagement() {
    // Cache invalidation strategy
    this.cacheInvalidationRules = {
      tournaments: 5 * 60 * 1000,    // 5 minutes
      clubs: 10 * 60 * 1000,        // 10 minutes
      venues: 15 * 60 * 1000,       // 15 minutes
      feedPosts: 2 * 60 * 1000,     // 2 minutes
      userProfile: 30 * 60 * 1000   // 30 minutes
    };
  }

  reducer(state, action) {
    switch (action.type) {
      case 'SET_PAGE':
        return { ...state, currentPage: action.payload };

      case 'SET_LOADING':
        return { ...state, isLoading: action.payload };

      case 'SET_MOBILE':
        return { ...state, isMobile: action.payload };

      case 'SET_USER':
        return {
          ...state,
          currentUser: action.payload,
          isAuthenticated: !!action.payload
        };

      case 'SET_TOURNAMENTS':
        return {
          ...state,
          tournaments: action.payload,
          lastUpdated: { ...state.lastUpdated, tournaments: Date.now() }
        };

      case 'ADD_TOURNAMENT':
        return {
          ...state,
          tournaments: [action.payload, ...state.tournaments]
        };

      case 'UPDATE_TOURNAMENT':
        return {
          ...state,
          tournaments: state.tournaments.map(t =>
            t.id === action.payload.id ? action.payload : t
          )
        };

      case 'SET_FEED_POSTS':
        return {
          ...state,
          feedPosts: action.payload,
          lastUpdated: { ...state.lastUpdated, feedPosts: Date.now() }
        };

      case 'ADD_FEED_POST':
        return {
          ...state,
          feedPosts: [action.payload, ...state.feedPosts]
        };

      case 'SHOW_MODAL':
        return {
          ...state,
          modals: {
            ...state.modals,
            [action.payload.type]: {
              isOpen: true,
              data: action.payload.data
            }
          }
        };

      case 'HIDE_MODAL':
        return {
          ...state,
          modals: {
            ...state.modals,
            [action.payload]: {
              isOpen: false,
              data: null
            }
          }
        };

      case 'CACHE_SET':
        state.cache.set(action.payload.key, {
          data: action.payload.data,
          timestamp: Date.now()
        });
        return state;

      case 'CACHE_CLEAR':
        if (action.payload) {
          state.cache.delete(action.payload);
        } else {
          state.cache.clear();
        }
        return state;

      default:
        return state;
    }
  }

  // Cache management methods
  getCachedData(key) {
    const cached = this.state.cache.get(key);
    if (!cached) return null;

    const now = Date.now();
    const maxAge = this.cacheInvalidationRules[key] || 5 * 60 * 1000;

    if (now - cached.timestamp > maxAge) {
      this.state.cache.delete(key);
      return null;
    }

    return cached.data;
  }

  setCachedData(key, data) {
    this.dispatch({
      type: 'CACHE_SET',
      payload: { key, data }
    });
  }

  clearCache(key = null) {
    this.dispatch({
      type: 'CACHE_CLEAR',
      payload: key
    });
  }

  // Data fetching with cache
  async getTournaments(force = false) {
    if (!force) {
      const cached = this.getCachedData('tournaments');
      if (cached) return cached;
    }

    this.dispatch({ type: 'SET_LOADING', payload: true });

    try {
      const response = await fetch('/api/v1/tournaments');
      const data = await response.json();

      if (data.success) {
        this.dispatch({ type: 'SET_TOURNAMENTS', payload: data.data.tournaments });
        this.setCachedData('tournaments', data.data.tournaments);
        return data.data.tournaments;
      }
    } catch (error) {
      console.error('Failed to fetch tournaments:', error);
    } finally {
      this.dispatch({ type: 'SET_LOADING', payload: false });
    }
  }
}

// Utility functions
function debounce(func, wait) {
  let timeout;
  return function executedFunction(...args) {
    const later = () => {
      clearTimeout(timeout);
      func(...args);
    };
    clearTimeout(timeout);
    timeout = setTimeout(later, wait);
  };
}

// Create global store instance
const appStore = new AppStore();
```

### Store Subscriptions and Middleware
```javascript
// src/Frontend/stores/middleware.js
class StoreMiddleware {
  static loggingMiddleware(store) {
    return (action) => {
      console.log('Action:', action.type, action.payload);
      console.log('Previous State:', store.getState());
      return action;
    };
  }

  static persistenceMiddleware(store) {
    return (action) => {
      // Persist certain state to localStorage
      const persistKeys = ['currentUser', 'currentPage'];

      if (persistKeys.some(key => action.type.includes(key.toUpperCase()))) {
        const state = store.getState();
        persistKeys.forEach(key => {
          if (state[key] !== undefined) {
            localStorage.setItem(`pingpong_${key}`, JSON.stringify(state[key]));
          }
        });
      }

      return action;
    };
  }

  static asyncMiddleware(store) {
    return async (action) => {
      if (action.type.endsWith('_ASYNC')) {
        const syncAction = { ...action, type: action.type.replace('_ASYNC', '') };

        try {
          store.dispatch({ type: 'SET_LOADING', payload: true });
          const result = await action.payload();
          store.dispatch({ ...syncAction, payload: result });
        } catch (error) {
          store.dispatch({
            type: `${syncAction.type}_ERROR`,
            payload: error.message
          });
        } finally {
          store.dispatch({ type: 'SET_LOADING', payload: false });
        }
      } else {
        return action;
      }
    };
  }
}

// Apply middleware to store
appStore.addMiddleware(StoreMiddleware.loggingMiddleware);
appStore.addMiddleware(StoreMiddleware.persistenceMiddleware);
appStore.addMiddleware(StoreMiddleware.asyncMiddleware);
```

## 2. Responsive UI Component Library

### Base Component System
```javascript
// src/Frontend/components/BaseComponent.js
export class BaseComponent extends HTMLElement {
  constructor() {
    super();
    this.state = {};
    this.props = {};
    this.eventListeners = [];
    this.shadow = this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.render();
    this.attachEventListeners();
    this.componentDidMount();
  }

  disconnectedCallback() {
    this.detachEventListeners();
    this.componentWillUnmount();
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue !== newValue) {
      this.props[name] = this.parseAttribute(newValue);
      this.render();
    }
  }

  setState(newState) {
    this.state = { ...this.state, ...newState };
    this.render();
  }

  setProps(newProps) {
    this.props = { ...this.props, ...newProps };
    this.render();
  }

  render() {
    // To be implemented by subclasses
    this.shadow.innerHTML = this.renderTemplate();
  }

  renderTemplate() {
    return '<div>Component template</div>';
  }

  attachEventListeners() {
    // Attach event listeners with cleanup tracking
  }

  detachEventListeners() {
    this.eventListeners.forEach(({ element, event, handler }) => {
      element.removeEventListener(event, handler);
    });
    this.eventListeners = [];
  }

  addEventListenerWithCleanup(element, event, handler) {
    element.addEventListener(event, handler);
    this.eventListeners.push({ element, event, handler });
  }

  parseAttribute(value) {
    try {
      return JSON.parse(value);
    } catch {
      return value;
    }
  }

  componentDidMount() {
    // Lifecycle method
  }

  componentWillUnmount() {
    // Lifecycle method
  }

  emit(eventName, detail = {}) {
    this.dispatchEvent(new CustomEvent(eventName, {
      detail,
      bubbles: true,
      composed: true
    }));
  }
}

// Register base component
customElements.define('base-component', BaseComponent);
```

### Core UI Components
```javascript
// src/Frontend/components/Button.js
export class Button extends BaseComponent {
  static get observedAttributes() {
    return ['variant', 'size', 'disabled', 'loading'];
  }

  constructor() {
    super();
    this.variant = 'primary';
    this.size = 'medium';
    this.disabled = false;
    this.loading = false;
  }

  renderTemplate() {
    const classes = this.getButtonClasses();
    const content = this.loading ? this.renderLoading() : this.textContent || 'Button';

    return `
      <style>
        .btn {
          display: inline-flex;
          align-items: center;
          justify-content: center;
          font-weight: 500;
          border-radius: 0.375rem;
          transition: all 0.2s;
          cursor: pointer;
          border: none;
          outline: none;
        }

        .btn:disabled {
          opacity: 0.5;
          cursor: not-allowed;
        }

        .btn-primary {
          background: #f97316;
          color: white;
        }

        .btn-primary:hover:not(:disabled) {
          background: #ea580c;
        }

        .btn-secondary {
          background: #f3f4f6;
          color: #374151;
        }

        .btn-secondary:hover:not(:disabled) {
          background: #e5e7eb;
        }

        .btn-small {
          padding: 0.375rem 0.75rem;
          font-size: 0.875rem;
        }

        .btn-medium {
          padding: 0.5rem 1rem;
          font-size: 1rem;
        }

        .btn-large {
          padding: 0.75rem 1.5rem;
          font-size: 1.125rem;
        }
      </style>
      <button class="btn ${classes}" ?disabled="${this.disabled}">
        ${content}
      </button>
    `;
  }

  getButtonClasses() {
    const classes = [];

    classes.push(`btn-${this.variant}`);
    classes.push(`btn-${this.size}`);

    if (this.disabled) classes.push('disabled');

    return classes.join(' ');
  }

  renderLoading() {
    return `
      <svg class="animate-spin -ml-1 mr-2 h-4 w-4" fill="none" viewBox="0 0 24 24">
        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"/>
        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"/>
      </svg>
      Loading...
    `;
  }

  attachEventListeners() {
    const button = this.shadow.querySelector('button');
    if (button) {
      this.addEventListenerWithCleanup(button, 'click', (e) => {
        if (!this.disabled && !this.loading) {
          this.emit('button-click', { originalEvent: e });
        }
      });
    }
  }
}

customElements.define('pp-button', Button);
```

```javascript
// src/Frontend/components/Modal.js
export class Modal extends BaseComponent {
  static get observedAttributes() {
    return ['open', 'title', 'size'];
  }

  constructor() {
    super();
    this.open = false;
    this.title = '';
    this.size = 'medium';
  }

  renderTemplate() {
    if (!this.open) return '';

    const sizeClasses = this.getSizeClasses();

    return `
      <style>
        .modal-overlay {
          position: fixed;
          top: 0;
          left: 0;
          right: 0;
          bottom: 0;
          background: rgba(0, 0, 0, 0.5);
          display: flex;
          align-items: center;
          justify-content: center;
          z-index: 1000;
        }

        .modal-content {
          background: white;
          border-radius: 0.5rem;
          box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
          max-height: 90vh;
          overflow-y: auto;
          margin: 1rem;
        }

        .modal-header {
          padding: 1.5rem;
          border-bottom: 1px solid #e5e7eb;
          display: flex;
          justify-content: space-between;
          align-items: center;
        }

        .modal-body {
          padding: 1.5rem;
        }

        .modal-footer {
          padding: 1rem 1.5rem;
          border-top: 1px solid #e5e7eb;
          display: flex;
          justify-content: flex-end;
          gap: 0.75rem;
        }

        .close-btn {
          color: #6b7280;
          hover:color: #374151;
          cursor: pointer;
        }

        .size-small { max-width: 20rem; }
        .size-medium { max-width: 32rem; }
        .size-large { max-width: 48rem; }
        .size-full { max-width: 90vw; }
      </style>

      <div class="modal-overlay">
        <div class="modal-content ${sizeClasses}">
          ${this.title ? `
            <div class="modal-header">
              <h3 class="text-lg font-semibold text-gray-900">${this.title}</h3>
              <button class="close-btn" data-action="close">
                <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
                </svg>
              </button>
            </div>
          ` : ''}

          <div class="modal-body">
            <slot></slot>
          </div>

          ${this.querySelector('[slot="footer"]') ? `
            <div class="modal-footer">
              <slot name="footer"></slot>
            </div>
          ` : ''}
        </div>
      </div>
    `;
  }

  getSizeClasses() {
    const sizes = {
      small: 'size-small',
      medium: 'size-medium',
      large: 'size-large',
      full: 'size-full'
    };
    return sizes[this.size] || sizes.medium;
  }

  attachEventListeners() {
    const overlay = this.shadow.querySelector('.modal-overlay');
    const closeBtn = this.shadow.querySelector('[data-action="close"]');

    if (overlay) {
      this.addEventListenerWithCleanup(overlay, 'click', (e) => {
        if (e.target === overlay) {
          this.close();
        }
      });
    }

    if (closeBtn) {
      this.addEventListenerWithCleanup(closeBtn, 'click', () => {
        this.close();
      });
    }

    // Handle escape key
    this.addEventListenerWithCleanup(document, 'keydown', (e) => {
      if (e.key === 'Escape' && this.open) {
        this.close();
      }
    });
  }

  open() {
    this.open = true;
    this.render();
    document.body.style.overflow = 'hidden';
    this.emit('modal-open');
  }

  close() {
    this.open = false;
    this.render();
    document.body.style.overflow = '';
    this.emit('modal-close');
  }

  attributeChangedCallback(name, oldValue, newValue) {
    super.attributeChangedCallback(name, oldValue, newValue);

    if (name === 'open') {
      if (newValue === 'true' || newValue === '') {
        this.open = true;
        document.body.style.overflow = 'hidden';
      } else {
        this.open = false;
        document.body.style.overflow = '';
      }
    }
  }
}

customElements.define('pp-modal', Modal);
```

```javascript
// src/Frontend/components/FormField.js
export class FormField extends BaseComponent {
  static get observedAttributes() {
    return ['type', 'label', 'placeholder', 'value', 'error', 'required', 'disabled'];
  }

  constructor() {
    super();
    this.type = 'text';
    this.label = '';
    this.placeholder = '';
    this.value = '';
    this.error = '';
    this.required = false;
    this.disabled = false;
  }

  renderTemplate() {
    return `
      <style>
        .form-field {
          margin-bottom: 1rem;
        }

        .form-label {
          display: block;
          font-size: 0.875rem;
          font-weight: 500;
          color: #374151;
          margin-bottom: 0.5rem;
        }

        .form-input {
          width: 100%;
          padding: 0.75rem;
          border: 1px solid #d1d5db;
          border-radius: 0.375rem;
          font-size: 1rem;
          transition: border-color 0.2s, box-shadow 0.2s;
        }

        .form-input:focus {
          outline: none;
          border-color: #f97316;
          box-shadow: 0 0 0 3px rgba(249, 115, 22, 0.1);
        }

        .form-input.error {
          border-color: #ef4444;
        }

        .form-input:disabled {
          background-color: #f9fafb;
          color: #6b7280;
          cursor: not-allowed;
        }

        .form-error {
          margin-top: 0.25rem;
          font-size: 0.875rem;
          color: #ef4444;
        }

        .required::after {
          content: ' *';
          color: #ef4444;
        }
      </style>

      <div class="form-field">
        ${this.label ? `<label class="form-label${this.required ? ' required' : ''}">${this.label}</label>` : ''}

        ${this.type === 'textarea' ? this.renderTextarea() : this.renderInput()}

        ${this.error ? `<div class="form-error">${this.error}</div>` : ''}
      </div>
    `;
  }

  renderInput() {
    return `
      <input
        type="${this.type}"
        value="${this.value}"
        placeholder="${this.placeholder}"
        class="form-input${this.error ? ' error' : ''}"
        ${this.required ? 'required' : ''}
        ${this.disabled ? 'disabled' : ''}
      >
    `;
  }

  renderTextarea() {
    return `
      <textarea
        placeholder="${this.placeholder}"
        class="form-input${this.error ? ' error' : ''}"
        rows="4"
        ${this.required ? 'required' : ''}
        ${this.disabled ? 'disabled' : ''}
      >${this.value}</textarea>
    `;
  }

  attachEventListeners() {
    const input = this.shadow.querySelector('.form-input');

    if (input) {
      this.addEventListenerWithCleanup(input, 'input', (e) => {
        this.value = e.target.value;
        this.emit('field-change', {
          value: this.value,
          field: this.getAttribute('name') || 'unnamed'
        });
      });

      this.addEventListenerWithCleanup(input, 'blur', (e) => {
        this.emit('field-blur', {
          value: this.value,
          field: this.getAttribute('name') || 'unnamed'
        });
      });
    }
  }

  setError(error) {
    this.error = error;
    this.render();
  }

  clearError() {
    this.error = '';
    this.render();
  }

  focus() {
    const input = this.shadow.querySelector('.form-input');
    if (input) input.focus();
  }
}

customElements.define('pp-form-field', FormField);
```

## 3. Build and Deployment Pipeline

### Vite Configuration
```javascript
// vite.config.js
import { defineConfig } from 'vite';
import { resolve } from 'path';
import { createHtmlPlugin } from 'vite-plugin-html';

export default defineConfig(({ mode }) => ({
  root: 'src/Frontend',
  publicDir: '../../../public',

  build: {
    outDir: '../../../public/assets',
    assetsDir: '',
    emptyOutDir: true,
    rollupOptions: {
      input: {
        main: resolve(__dirname, 'src/Frontend/index.html'),
        styles: resolve(__dirname, 'src/Frontend/styles/main.css')
      },
      output: {
        entryFileNames: 'js/[name].[hash].js',
        chunkFileNames: 'js/[name].[hash].js',
        assetFileNames: (assetInfo) => {
          if (assetInfo.name?.endsWith('.css')) {
            return 'css/[name].[hash][extname]';
          }
          return 'assets/[name].[hash][extname]';
        }
      }
    },
    minify: mode === 'production' ? 'terser' : false,
    sourcemap: mode === 'development'
  },

  server: {
    host: '0.0.0.0',
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        secure: false
      }
    },
    cors: true
  },

  plugins: [
    createHtmlPlugin({
      minify: mode === 'production',
      inject: {
        data: {
          title: 'PingPong+',
          description: 'Table Tennis Social Platform'
        }
      }
    })
  ],

  resolve: {
    alias: {
      '@': resolve(__dirname, 'src/Frontend'),
      '@components': resolve(__dirname, 'src/Frontend/components'),
      '@pages': resolve(__dirname, 'src/Frontend/pages'),
      '@services': resolve(__dirname, 'src/Frontend/services'),
      '@stores': resolve(__dirname, 'src/Frontend/stores'),
      '@utils': resolve(__dirname, 'src/Frontend/utils'),
      '@styles': resolve(__dirname, 'src/Frontend/styles')
    }
  },

  css: {
    postcss: './postcss.config.js',
    modules: {
      localsConvention: 'camelCase'
    }
  },

  optimizeDeps: {
    include: ['tailwindcss', 'lucide']
  }
}));
```

### Package.json Scripts
```json
{
  "name": "pingpong-plus-frontend",
  "version": "1.0.0",
  "description": "Frontend for PingPong+ table tennis social platform",
  "main": "src/Frontend/index.js",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint src/Frontend --ext .js,.jsx,.ts,.tsx",
    "lint:fix": "eslint src/Frontend --ext .js,.jsx,.ts,.tsx --fix",
    "format": "prettier --write src/Frontend/**/*.{js,jsx,ts,tsx,css,md}",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "type-check": "tsc --noEmit",
    "analyze": "vite build --mode analyze"
  },
  "dependencies": {
    "lucide": "^0.263.0"
  },
  "devDependencies": {
    "@vitejs/plugin-legacy": "^4.0.1",
    "autoprefixer": "^10.4.14",
    "eslint": "^8.45.0",
    "jest": "^29.6.1",
    "postcss": "^8.4.24",
    "prettier": "^3.0.0",
    "tailwindcss": "^3.3.2",
    "typescript": "^5.1.6",
    "vite": "^4.4.0",
    "vite-plugin-html": "^3.2.0"
  }
}
```

### Docker Configuration
```dockerfile
# Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production stage
FROM nginx:alpine

# Copy built assets
COPY --from=builder /app/public /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:80"
    environment:
      - NODE_ENV=production
    depends_on:
      - api
    networks:
      - pingpong-network

  api:
    build: ./src/Backend
    ports:
      - "8000:8000"
    environment:
      - DB_HOST=db
      - DB_PORT=3306
      - DB_NAME=pingpong
      - DB_USER=pingpong_user
      - DB_PASSWORD=pingpong_pass
    depends_on:
      - db
    networks:
      - pingpong-network

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=root_password
      - MYSQL_DATABASE=pingpong
      - MYSQL_USER=pingpong_user
      - MYSQL_PASSWORD=pingpong_pass
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      - ./src/Database/schema.sql:/docker-entrypoint-initdb.d/schema.sql
    networks:
      - pingpong-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - pingpong-network

networks:
  pingpong-network:
    driver: bridge

volumes:
  db_data:
```

### CI/CD Pipeline
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run linting
      run: npm run lint

    - name: Run tests
      run: npm run test

    - name: Build application
      run: npm run build

    - name: Upload build artifacts
      uses: actions/upload-artifact@v3
      with:
        name: build-files
        path: public/

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - name: Download build artifacts
      uses: actions/download-artifact@v3
      with:
        name: build-files
        path: public/

    - name: Deploy to server
      uses: appleboy/ssh-action@v0.1.10
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SERVER_KEY }}
        script: |
          cd /var/www/pingpong-plus
          git pull origin main
          npm ci
          npm run build
          sudo systemctl restart nginx
```

## 4. Development Environment Setup

### Local Development Setup
```bash
#!/bin/bash
# setup.sh

echo "Setting up PingPong+ development environment..."

# Check prerequisites
command -v php >/dev/null 2>&1 || { echo "PHP is required but not installed. Aborting." >&2; exit 1; }
command -v composer >/dev/null 2>&1 || { echo "Composer is required but not installed. Aborting." >&2; exit 1; }
command -v node >/dev/null 2>&1 || { echo "Node.js is required but not installed. Aborting." >&2; }
command -v npm >/dev/null 2>&1 || { echo "npm is required but not installed. Aborting." >&2; }

# Backend setup
echo "Setting up backend..."
cd src/Backend
composer install
cp .env.example .env
# Configure database settings in .env

# Frontend setup
echo "Setting up frontend..."
cd ../Frontend
npm install
cp .env.example .env
# Configure API endpoints in .env

# Database setup
echo "Setting up database..."
cd ../Database
# Run migrations and seeders

echo "Development environment setup complete!"
echo "Run 'npm run dev' in frontend directory to start development server"
echo "Run 'php -S localhost:8000' in backend directory to start API server"
```

### Environment Configuration
```bash
# .env.example (Backend)
APP_NAME="PingPong+"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=pingpong_plus
DB_USERNAME=root
DB_PASSWORD=

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

JWT_SECRET=your-super-secret-jwt-key-here
JWT_TTL=3600

MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS="noreply@pingpongplus.com"
MAIL_FROM_NAME="${APP_NAME}"

# .env.example (Frontend)
VITE_API_BASE_URL=http://localhost:8000/api/v1
VITE_APP_NAME="PingPong+"
VITE_APP_VERSION="1.0.0"
VITE_GOOGLE_MAPS_API_KEY=your-google-maps-api-key
VITE_SENTRY_DSN=your-sentry-dsn
```

## 5. Testing Strategy

### Backend Testing
```php
<?php
// tests/Backend/Unit/UserTest.php
use PHPUnit\Framework\TestCase;
use App\Models\User;
use App\Services\UserService;

class UserTest extends TestCase
{
    private $userService;
    private $pdo;

    protected function setUp(): void
    {
        // Set up in-memory SQLite database for testing
        $this->pdo = new PDO('sqlite::memory:');
        $this->pdo->exec(file_get_contents(__DIR__ . '/../../src/Database/schema.sql'));

        $this->userService = new UserService($this->pdo);
    }

    public function testCreateUser()
    {
        $userData = [
            'name' => 'Test User',
            'email' => 'test@example.com',
            'password' => 'password123'
        ];

        $userId = $this->userService->createUser($userData);

        $this->assertIsInt($userId);
        $this->assertGreaterThan(0, $userId);
    }

    public function testFindUserByEmail()
    {
        $userData = [
            'name' => 'Test User',
            'email' => 'test@example.com',
            'password' => 'password123'
        ];

        $this->userService->createUser($userData);
        $user = $this->userService->findUserByEmail('test@example.com');

        $this->assertIsArray($user);
        $this->assertEquals('Test User', $user['name']);
        $this->assertEquals('test@example.com', $user['email']);
    }

    public function testAuthenticateUser()
    {
        $userData = [
            'name' => 'Test User',
            'email' => 'test@example.com',
            'password' => 'password123'
        ];

        $this->userService->createUser($userData);
        $authenticatedUser = $this->userService->authenticate('test@example.com', 'password123');

        $this->assertIsArray($authenticatedUser);
        $this->assertEquals('Test User', $authenticatedUser['name']);
    }

    public function testAuthenticateUserWithWrongPassword()
    {
        $userData = [
            'name' => 'Test User',
            'email' => 'test@example.com',
            'password' => 'password123'
        ];

        $this->userService->createUser($userData);
        $authenticatedUser = $this->userService->authenticate('test@example.com', 'wrongpassword');

        $this->assertNull($authenticatedUser);
    }
}
```

```php
<?php
// tests/Backend/Integration/TournamentApiTest.php
use PHPUnit\Framework\TestCase;

class TournamentApiTest extends TestCase
{
    private $client;
    private $testUser;

    protected function setUp(): void
    {
        $this->client = new GuzzleHttp\Client([
            'base_uri' => 'http://localhost:8000/api/v1/',
            'http_errors' => false
        ]);

        // Create test user and get auth token
        $this->testUser = $this->createTestUser();
    }

    public function testCreateTournament()
    {
        $tournamentData = [
            'name' => 'Test Tournament',
            'description' => 'Test tournament description',
            'date' => '2025-09-01',
            'location' => 'Test Venue',
            'category' => 'Singles • Menengah',
            'max_participants' => 16,
            'fee' => 50000,
            'prize' => 1000000
        ];

        $response = $this->client->post('tournaments', [
            'headers' => [
                'Authorization' => 'Bearer ' . $this->testUser['token'],
                'Content-Type' => 'application/json'
            ],
            'json' => $tournamentData
        ]);

        $this->assertEquals(201, $response->getStatusCode());

        $data = json_decode($response->getBody(), true);
        $this->assertTrue($data['success']);
        $this->assertArrayHasKey('tournament_id', $data['data']);
    }

    public function testGetTournaments()
    {
        $response = $this->client->get('tournaments');

        $this->assertEquals(200, $response->getStatusCode());

        $data = json_decode($response->getBody(), true);
        $this->assertTrue($data['success']);
        $this->assertArrayHasKey('tournaments', $data['data']);
        $this->assertIsArray($data['data']['tournaments']);
    }

    public function testRegisterForTournament()
    {
        // First create a tournament
        $tournamentData = [
            'name' => 'Registration Test Tournament',
            'description' => 'Test tournament for registration',
            'date' => '2025-09-01',
            'location' => 'Test Venue',
            'category' => 'Singles • Menengah',
            'max_participants' => 16,
            'fee' => 50000,
            'prize' => 1000000
        ];

        $createResponse = $this->client->post('tournaments', [
            'headers' => [
                'Authorization' => 'Bearer ' . $this->testUser['token'],
                'Content-Type' => 'application/json'
            ],
            'json' => $tournamentData
        ]);

        $createData = json_decode($createResponse->getBody(), true);
        $tournamentId = $createData['data']['tournament_id'];

        // Now register for the tournament
        $response = $this->client->post("tournaments/{$tournamentId}/register", [
            'headers' => [
                'Authorization' => 'Bearer ' . $this->testUser['token']
            ]
        ]);

        $this->assertEquals(200, $response->getStatusCode());

        $data = json_decode($response->getBody(), true);
        $this->assertTrue($data['success']);
    }

    private function createTestUser()
    {
        // Create a test user and return auth token
        // Implementation depends on your user creation logic
        return [
            'id' => 1,
            'token' => 'test_jwt_token'
        ];
    }
}
```

### Frontend Testing
```javascript
// tests/Frontend/components/Button.test.js
import { Button } from '../../src/Frontend/components/Button.js';

describe('Button Component', () => {
  let button;
  let container;

  beforeEach(() => {
    container = document.createElement('div');
    document.body.appendChild(container);
    button = new Button();
    container.appendChild(button);
  });

  afterEach(() => {
    document.body.removeChild(container);
  });

  test('renders with default props', () => {
    expect(button.shadowRoot.querySelector('button')).toBeTruthy();
    expect(button.shadowRoot.textContent).toContain('Button');
  });

  test('renders with custom text', () => {
    button.textContent = 'Click Me';
    button.render();

    expect(button.shadowRoot.textContent).toContain('Click Me');
  });

  test('applies primary variant classes', () => {
    button.setAttribute('variant', 'primary');
    button.render();

    const buttonElement = button.shadowRoot.querySelector('button');
    expect(buttonElement.classList.contains('btn-primary')).toBe(true);
  });

  test('handles click events', () => {
    const mockHandler = jest.fn();
    button.addEventListener('button-click', mockHandler);

    const buttonElement = button.shadowRoot.querySelector('button');
    buttonElement.click();

    expect(mockHandler).toHaveBeenCalled();
  });

  test('disables button when disabled attribute is set', () => {
    button.setAttribute('disabled', '');
    button.render();

    const buttonElement = button.shadowRoot.querySelector('button');
    expect(buttonElement.disabled).toBe(true);
  });

  test('shows loading state', () => {
    button.setAttribute('loading', '');
    button.render();

    expect(button.shadowRoot.textContent).toContain('Loading...');
  });
});
```

```javascript
// tests/Frontend/services/ApiService.test.js
import { ApiService } from '../../src/Frontend/services/ApiService.js';

// Mock fetch
global.fetch = jest.fn();

describe('ApiService', () => {
  let apiService;

  beforeEach(() => {
    apiService = new ApiService();
    fetch.mockClear();
  });

  test('makes GET request', async () => {
    const mockResponse = {
      success: true,
      data: { message: 'Hello World' }
    };

    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockResponse)
    });

    const result = await apiService.get('/test');

    expect(fetch).toHaveBeenCalledWith('/api/v1/test', {
      method: 'GET',
      headers: { 'Content-Type': 'application/json' }
    });

    expect(result).toEqual(mockResponse);
  });

  test('makes POST request with data', async () => {
    const mockResponse = {
      success: true,
      data: { id: 1 }
    };

    const postData = { name: 'Test' };

    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockResponse)
    });

    const result = await apiService.post('/test', postData);

    expect(fetch).toHaveBeenCalledWith('/api/v1/test', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(postData)
    });

    expect(result).toEqual(mockResponse);
  });

  test('handles HTTP errors', async () => {
    fetch.mockResolvedValueOnce({
      ok: false,
      status: 404,
      json: () => Promise.resolve({ error: 'Not found' })
    });

    await expect(apiService.get('/notfound')).rejects.toThrow('Not found');
  });

  test('adds authorization header when token is available', async () => {
    localStorage.setItem('auth_token', 'test_token');

    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ success: true })
    });

    await apiService.get('/protected');

    expect(fetch).toHaveBeenCalledWith('/api/v1/protected', {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer test_token'
      }
    });

    localStorage.removeItem('auth_token');
  });
});
```

### E2E Testing
```javascript
// tests/Frontend/e2e/tournament-creation.spec.js
import { test, expect } from '@playwright/test';

test.describe('Tournament Creation', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    await page.goto('/login');
    await page.fill('[data-testid="email"]', 'test@example.com');
    await page.fill('[data-testid="password"]', 'password123');
    await page.click('[data-testid="login-button"]');
    await page.waitForURL('/home');
  });

  test('creates a new tournament successfully', async ({ page }) => {
    // Navigate to tournaments page
    await page.click('[data-testid="nav-tournaments"]');
    await page.waitForURL('/tournaments');

    // Click create tournament button
    await page.click('[data-testid="create-tournament-btn"]');

    // Fill tournament form
    await page.fill('[data-testid="tournament-name"]', 'E2E Test Tournament');
    await page.fill('[data-testid="tournament-description"]', 'Test tournament created by E2E test');
    await page.fill('[data-testid="tournament-date"]', '2025-12-01');
    await page.fill('[data-testid="tournament-location"]', 'Test Venue');
    await page.selectOption('[data-testid="tournament-category"]', 'Singles • Menengah');
    await page.fill('[data-testid="max-participants"]', '16');
    await page.fill('[data-testid="entry-fee"]', '75000');

    // Submit form
    await page.click('[data-testid="submit-tournament"]');

    // Verify success message
    await expect(page.locator('[data-testid="success-message"]')).toContainText('Tournament created successfully');

    // Verify tournament appears in list
    await page.waitForSelector('[data-testid="tournament-list"]');
    await expect(page.locator('[data-testid="tournament-list"]')).toContainText('E2E Test Tournament');
  });

  test('validates required fields', async ({ page }) => {
    await page.click('[data-testid="nav-tournaments"]');
    await page.click('[data-testid="create-tournament-btn"]');

    // Try to submit without filling required fields
    await page.click('[data-testid="submit-tournament"]');

    // Check for validation errors
    await expect(page.locator('[data-testid="name-error"]')).toContainText('Name is required');
    await expect(page.locator('[data-testid="date-error"]')).toContainText('Date is required');
    await expect(page.locator('[data-testid="location-error"]')).toContainText('Location is required');
  });

  test('prevents creation with insufficient permissions', async ({ page }) => {
    // This test would require a user with basic tier
    await page.click('[data-testid="nav-tournaments"]');

    // Create tournament button should not be visible or disabled
    await expect(page.locator('[data-testid="create-tournament-btn"]')).not.toBeVisible();
  });
});
```

## 6. Security Implementation

### Input Validation and Sanitization
```php
<?php
// src/Backend/Utils/Security.php
class Security {
    public static function sanitizeInput($input) {
        if (is_array($input)) {
            return array_map([self::class, 'sanitizeInput'], $input);
        }

        if (is_string($input)) {
            // Remove null bytes
            $input = str_replace(chr(0), '', $input);

            // Convert special characters to HTML entities
            $input = htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF