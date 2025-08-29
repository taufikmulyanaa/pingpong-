# PingPong+ Frontend Architecture

## Overview
Modern, component-based frontend architecture using vanilla JavaScript with a focus on maintainability, performance, and user experience.

## Technology Stack

### Core Technologies
- **JavaScript**: ES6+ with modules
- **HTML5**: Semantic markup
- **CSS3**: Tailwind CSS for styling
- **Build Tool**: Vite for development and bundling
- **Package Manager**: npm

### Development Tools
- **Linting**: ESLint with custom rules
- **Formatting**: Prettier
- **Testing**: Jest for unit tests, Cypress for E2E
- **Documentation**: JSDoc

## Architecture Patterns

### Component-Based Architecture
```javascript
// Base Component Class
class Component {
  constructor(props = {}) {
    this.props = props;
    this.state = {};
    this.element = null;
    this.eventListeners = [];
  }

  setState(newState) {
    this.state = { ...this.state, ...newState };
    this.render();
    this.notifyStateChange();
  }

  render() {
    // Virtual method - to be implemented by subclasses
    throw new Error('Component must implement render() method');
  }

  mount(container) {
    this.element = this.render();
    container.appendChild(this.element);
    this.attachEventListeners();
    this.componentDidMount();
  }

  unmount() {
    if (this.element && this.element.parentNode) {
      this.detachEventListeners();
      this.element.parentNode.removeChild(this.element);
      this.componentWillUnmount();
    }
  }

  attachEventListeners() {
    // Attach event listeners and store references for cleanup
  }

  detachEventListeners() {
    this.eventListeners.forEach(({ element, event, handler }) => {
      element.removeEventListener(event, handler);
    });
    this.eventListeners = [];
  }

  componentDidMount() {
    // Lifecycle method
  }

  componentWillUnmount() {
    // Lifecycle method
  }

  notifyStateChange() {
    // Notify parent components or stores of state changes
  }
}
```

### State Management Pattern
```javascript
// Store Pattern Implementation
class Store {
  constructor(initialState = {}) {
    this.state = { ...initialState };
    this.listeners = new Set();
    this.middlewares = [];
  }

  subscribe(listener) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  dispatch(action) {
    // Apply middlewares
    let processedAction = action;
    for (const middleware of this.middlewares) {
      processedAction = middleware(processedAction, this.state);
    }

    // Update state
    this.state = this.reducer(this.state, processedAction);

    // Notify listeners
    this.listeners.forEach(listener => listener(this.state));
  }

  getState() {
    return { ...this.state };
  }

  addMiddleware(middleware) {
    this.middlewares.push(middleware);
  }

  reducer(state, action) {
    // To be implemented by specific stores
    return state;
  }
}

// Example User Store
class UserStore extends Store {
  constructor() {
    super({
      currentUser: null,
      isAuthenticated: false,
      loading: false,
      error: null
    });
  }

  reducer(state, action) {
    switch (action.type) {
      case 'USER_LOGIN_START':
        return { ...state, loading: true, error: null };

      case 'USER_LOGIN_SUCCESS':
        return {
          ...state,
          loading: false,
          currentUser: action.payload.user,
          isAuthenticated: true,
          error: null
        };

      case 'USER_LOGIN_ERROR':
        return {
          ...state,
          loading: false,
          error: action.payload.error,
          isAuthenticated: false
        };

      case 'USER_LOGOUT':
        return {
          ...state,
          currentUser: null,
          isAuthenticated: false,
          error: null
        };

      default:
        return state;
    }
  }
}
```

### Service Layer Pattern
```javascript
// API Service Base Class
class ApiService {
  constructor(baseURL = '/api/v1') {
    this.baseURL = baseURL;
    this.defaultHeaders = {
      'Content-Type': 'application/json'
    };
  }

  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const config = {
      headers: { ...this.defaultHeaders, ...options.headers },
      ...options
    };

    // Add authentication token if available
    const token = this.getAuthToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    try {
      const response = await fetch(url, config);
      return await this.handleResponse(response);
    } catch (error) {
      return this.handleError(error);
    }
  }

  async get(endpoint, params = {}) {
    const queryString = new URLSearchParams(params).toString();
    const url = queryString ? `${endpoint}?${queryString}` : endpoint;
    return this.request(url, { method: 'GET' });
  }

  async post(endpoint, data = {}) {
    return this.request(endpoint, {
      method: 'POST',
      body: JSON.stringify(data)
    });
  }

  async put(endpoint, data = {}) {
    return this.request(endpoint, {
      method: 'PUT',
      body: JSON.stringify(data)
    });
  }

  async delete(endpoint) {
    return this.request(endpoint, { method: 'DELETE' });
  }

  getAuthToken() {
    return localStorage.getItem('auth_token');
  }

  async handleResponse(response) {
    if (!response.ok) {
      const error = await response.json().catch(() => ({ message: 'Network error' }));
      throw new Error(error.message || `HTTP ${response.status}`);
    }

    return await response.json();
  }

  handleError(error) {
    console.error('API Error:', error);
    throw error;
  }
}

// Specific Service Classes
class UserService extends ApiService {
  async getProfile(userId = null) {
    const endpoint = userId ? `/users/${userId}` : '/users/profile';
    return this.get(endpoint);
  }

  async updateProfile(data) {
    return this.put('/users/profile', data);
  }

  async searchUsers(filters = {}) {
    return this.get('/users/search', filters);
  }
}

class TournamentService extends ApiService {
  async getTournaments(filters = {}) {
    return this.get('/tournaments', filters);
  }

  async getTournament(id) {
    return this.get(`/tournaments/${id}`);
  }

  async createTournament(data) {
    return this.post('/tournaments', data);
  }

  async registerForTournament(tournamentId) {
    return this.post(`/tournaments/${tournamentId}/register`);
  }
}
```

## Component Structure

### Page Components
```javascript
// Base Page Component
class PageComponent extends Component {
  constructor(props = {}) {
    super(props);
    this.loading = false;
    this.error = null;
  }

  async loadData() {
    // Virtual method for data loading
  }

  showLoading() {
    this.loading = true;
    this.render();
  }

  hideLoading() {
    this.loading = false;
    this.render();
  }

  showError(error) {
    this.error = error;
    this.render();
  }

  clearError() {
    this.error = null;
    this.render();
  }

  render() {
    if (this.loading) {
      return this.renderLoading();
    }

    if (this.error) {
      return this.renderError();
    }

    return this.renderContent();
  }

  renderLoading() {
    return `
      <div class="flex items-center justify-center p-8">
        <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-orange-500"></div>
        <span class="ml-2 text-gray-600">Loading...</span>
      </div>
    `;
  }

  renderError() {
    return `
      <div class="bg-red-50 border border-red-200 rounded-lg p-4">
        <div class="flex items-center">
          <svg class="w-5 h-5 text-red-500 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
          </svg>
          <span class="text-red-700">${this.error}</span>
        </div>
        <button class="mt-2 text-red-600 hover:text-red-800 text-sm" onclick="this.clearError()">
          Try Again
        </button>
      </div>
    `;
  }

  renderContent() {
    // To be implemented by subclasses
    return '<div>Page content</div>';
  }
}

// Example Home Page
class HomePage extends PageComponent {
  constructor(props = {}) {
    super(props);
    this.feedPosts = [];
    this.userStats = null;
  }

  async loadData() {
    try {
      this.showLoading();

      const [feedResponse, userResponse] = await Promise.all([
        FeedService.getFeed({ limit: 20 }),
        UserService.getProfile()
      ]);

      this.feedPosts = feedResponse.data.posts;
      this.userStats = userResponse.data.user;

      this.hideLoading();
    } catch (error) {
      this.showError(error.message);
    }
  }

  renderContent() {
    return `
      <div class="space-y-6">
        ${this.renderUserStats()}
        ${this.renderFeed()}
      </div>
    `;
  }

  renderUserStats() {
    if (!this.userStats) return '';

    return `
      <div class="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden">
        <div class="bg-gradient-to-r from-orange-500 to-orange-600 p-6 text-white">
          <div class="flex items-center">
            <img src="${this.userStats.photo}" class="w-16 h-16 rounded-full border-4 border-white shadow-lg">
            <div class="ml-4">
              <h2 class="text-xl font-bold">${this.userStats.name}</h2>
              <p class="text-orange-100">⭐ ${this.userStats.elo} Rating</p>
            </div>
          </div>
        </div>
        <div class="p-6">
          <div class="grid grid-cols-4 gap-4">
            <div class="text-center">
              <p class="text-2xl font-bold text-gray-900">${this.userStats.matches}</p>
              <p class="text-sm text-gray-600">Matches</p>
            </div>
            <div class="text-center">
              <p class="text-2xl font-bold text-gray-900">${this.userStats.wins}</p>
              <p class="text-sm text-gray-600">Wins</p>
            </div>
            <div class="text-center">
              <p class="text-2xl font-bold text-gray-900">${this.userStats.win_rate}%</p>
              <p class="text-sm text-gray-600">Win Rate</p>
            </div>
            <div class="text-center">
              <p class="text-2xl font-bold text-gray-900">#${this.userStats.rank}</p>
              <p class="text-sm text-gray-600">Rank</p>
            </div>
          </div>
        </div>
      </div>
    `;
  }

  renderFeed() {
    if (!this.feedPosts.length) {
      return `
        <div class="bg-white rounded-xl shadow-sm border border-gray-200 p-8 text-center text-gray-500">
          No posts yet. Start following some players!
        </div>
      `;
    }

    return `
      <div class="bg-white rounded-xl shadow-sm border border-gray-200">
        <div class="p-4 border-b border-gray-200">
          <h3 class="text-lg font-semibold text-gray-900">Community Feed</h3>
        </div>
        <div class="divide-y divide-gray-200">
          ${this.feedPosts.map(post => this.renderFeedPost(post)).join('')}
        </div>
      </div>
    `;
  }

  renderFeedPost(post) {
    return `
      <div class="p-4">
        <div class="flex items-start space-x-3">
          <img src="${post.user_photo}" class="w-10 h-10 rounded-full">
          <div class="flex-1 min-w-0">
            <div class="flex items-center space-x-2 mb-2">
              <p class="font-medium text-gray-900">${post.user_name}</p>
              <span class="text-sm text-gray-500">${post.time_ago}</span>
            </div>
            <p class="text-gray-800 mb-3">${post.content}</p>
            ${post.image ? `<img src="${post.image}" class="w-full h-48 object-cover rounded-lg mb-3">` : ''}
            <div class="flex items-center space-x-4 text-sm text-gray-500">
              <button class="flex items-center space-x-1 hover:text-red-500">
                <span>❤️</span>
                <span>${post.likes}</span>
              </button>
              <button class="flex items-center space-x-1 hover:text-blue-500">
                <span>💬</span>
                <span>${post.comments}</span>
              </button>
            </div>
          </div>
        </div>
      </div>
    `;
  }
}
```

### Reusable UI Components
```javascript
// Button Component
class Button extends Component {
  constructor(props = {}) {
    super(props);
    this.variant = props.variant || 'primary';
    this.size = props.size || 'medium';
    this.disabled = props.disabled || false;
    this.loading = props.loading || false;
  }

  render() {
    const baseClasses = 'inline-flex items-center justify-center font-medium rounded-lg transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2';
    const variantClasses = this.getVariantClasses();
    const sizeClasses = this.getSizeClasses();
    const disabledClasses = this.disabled ? 'opacity-50 cursor-not-allowed' : 'hover:shadow-md';

    return `
      <button
        class="${baseClasses} ${variantClasses} ${sizeClasses} ${disabledClasses}"
        ${this.disabled ? 'disabled' : ''}
        ${this.props.onClick ? `onclick="${this.props.onClick}"` : ''}
      >
        ${this.loading ? this.renderLoadingSpinner() : ''}
        ${this.props.children || this.props.text || 'Button'}
      </button>
    `;
  }

  getVariantClasses() {
    const variants = {
      primary: 'bg-orange-500 text-white hover:bg-orange-600 focus:ring-orange-500',
      secondary: 'bg-gray-100 text-gray-700 hover:bg-gray-200 focus:ring-gray-500',
      danger: 'bg-red-500 text-white hover:bg-red-600 focus:ring-red-500',
      success: 'bg-green-500 text-white hover:bg-green-600 focus:ring-green-500'
    };
    return variants[this.variant] || variants.primary;
  }

  getSizeClasses() {
    const sizes = {
      small: 'px-3 py-1.5 text-sm',
      medium: 'px-4 py-2 text-base',
      large: 'px-6 py-3 text-lg'
    };
    return sizes[this.size] || sizes.medium;
  }

  renderLoadingSpinner() {
    return `
      <svg class="animate-spin -ml-1 mr-2 h-4 w-4" fill="none" viewBox="0 0 24 24">
        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"/>
        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"/>
      </svg>
    `;
  }
}

// Modal Component
class Modal extends Component {
  constructor(props = {}) {
    super(props);
    this.isOpen = props.isOpen || false;
    this.title = props.title || '';
    this.size = props.size || 'medium';
  }

  render() {
    if (!this.isOpen) return '';

    const sizeClasses = this.getSizeClasses();

    return `
      <div class="fixed inset-0 z-50 overflow-y-auto">
        <div class="flex items-center justify-center min-h-screen pt-4 px-4 pb-20 text-center sm:block sm:p-0">
          <div class="fixed inset-0 transition-opacity" aria-hidden="true">
            <div class="absolute inset-0 bg-gray-500 opacity-75" onclick="this.close()"></div>
          </div>

          <div class="inline-block align-bottom bg-white rounded-lg text-left overflow-hidden shadow-xl transform transition-all sm:my-8 sm:align-middle ${sizeClasses}">
            ${this.renderHeader()}
            <div class="px-4 pt-5 pb-4 bg-white sm:p-6 sm:pb-4">
              ${this.props.children || ''}
            </div>
            ${this.renderFooter()}
          </div>
        </div>
      </div>
    `;
  }

  renderHeader() {
    if (!this.title) return '';

    return `
      <div class="px-4 py-5 sm:px-6 bg-gray-50">
        <div class="flex items-center justify-between">
          <h3 class="text-lg leading-6 font-medium text-gray-900">${this.title}</h3>
          <button onclick="this.close()" class="text-gray-400 hover:text-gray-600">
            <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
            </svg>
          </button>
        </div>
      </div>
    `;
  }

  renderFooter() {
    if (!this.props.footer) return '';

    return `
      <div class="px-4 py-3 bg-gray-50 sm:px-6 sm:flex sm:flex-row-reverse">
        ${this.props.footer}
      </div>
    `;
  }

  getSizeClasses() {
    const sizes = {
      small: 'max-w-md',
      medium: 'max-w-lg',
      large: 'max-w-2xl',
      full: 'max-w-4xl'
    };
    return sizes[this.size] || sizes.medium;
  }

  open() {
    this.isOpen = true;
    this.render();
    document.body.classList.add('overflow-hidden');
  }

  close() {
    this.isOpen = false;
    this.render();
    document.body.classList.remove('overflow-hidden');
  }
}

// Form Components
class FormField extends Component {
  constructor(props = {}) {
    super(props);
    this.type = props.type || 'text';
    this.label = props.label || '';
    this.placeholder = props.placeholder || '';
    this.value = props.value || '';
    this.error = props.error || '';
    this.required = props.required || false;
  }

  render() {
    return `
      <div class="mb-4">
        ${this.label ? `<label class="block text-sm font-medium text-gray-700 mb-2">${this.label}</label>` : ''}
        <input
          type="${this.type}"
          value="${this.value}"
          placeholder="${this.placeholder}"
          class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-orange-500 focus:border-transparent ${this.error ? 'border-red-500' : ''}"
          ${this.required ? 'required' : ''}
        >
        ${this.error ? `<p class="mt-1 text-sm text-red-600">${this.error}</p>` : ''}
      </div>
    `;
  }
}
```

## Router Implementation

### SPA Router
```javascript
class Router {
  constructor() {
    this.routes = new Map();
    this.currentRoute = null;
    this.params = {};
    this.init();
  }

  init() {
    // Handle initial load
    this.handleRoute(window.location.pathname);

    // Handle browser back/forward
    window.addEventListener('popstate', (e) => {
      this.handleRoute(window.location.pathname, e.state);
    });

    // Handle link clicks
    document.addEventListener('click', (e) => {
      const link = e.target.closest('[data-route]');
      if (link) {
        e.preventDefault();
        const route = link.getAttribute('data-route');
        this.navigate(route);
      }
    });
  }

  addRoute(path, component) {
    // Convert route with params (e.g., /users/:id) to regex
    const paramNames = [];
    const regexPath = path.replace(/:([^\/]+)/g, (match, paramName) => {
      paramNames.push(paramName);
      return '([^\/]+)';
    });

    this.routes.set(path, {
      component,
      regex: new RegExp(`^${regexPath}$`),
      paramNames
    });
  }

  navigate(path, state = {}) {
    // Update URL without page reload
    window.history.pushState(state, '', path);
    this.handleRoute(path, state);
  }

  replace(path, state = {}) {
    window.history.replaceState(state, '', path);
    this.handleRoute(path, state);
  }

  handleRoute(path, state = {}) {
    for (const [routePath, routeConfig] of this.routes) {
      const match = path.match(routeConfig.regex);
      if (match) {
        // Extract parameters
        const params = {};
        routeConfig.paramNames.forEach((name, index) => {
          params[name] = match[index + 1];
        });

        this.params = params;
        this.currentRoute = routePath;

        // Render component
        this.renderRoute(routeConfig.component, params, state);
        break;
      }
    }
  }

  async renderRoute(ComponentClass, params, state) {
    // Clear current content
    const container = document.getElementById('app-container');
    container.innerHTML = '';

    // Create and render new component
    const component = new ComponentClass({ params, state });
    await component.loadData();
    component.mount(container);
  }

  getCurrentParams() {
    return { ...this.params };
  }

  getCurrentRoute() {
    return this.currentRoute;
  }
}

// Usage
const router = new Router();

// Add routes
router.addRoute('/', HomePage);
router.addRoute('/discover', DiscoverPage);
router.addRoute('/tournaments', TournamentsPage);
router.addRoute('/tournaments/:id', TournamentDetailPage);
router.addRoute('/clubs/:id', ClubDetailPage);
router.addRoute('/marketplace', MarketplacePage);
router.addRoute('/profile', ProfilePage);

// Navigation
router.navigate('/tournaments/123');
```

## State Management Implementation

### Global App Store
```javascript
// App Store
class AppStore extends Store {
  constructor() {
    super({
      currentPage: 'home',
      isMobile: false,
      loading: false,
      notifications: [],
      modal: {
        isOpen: false,
        type: null,
        data: null
      }
    });

    this.initResponsiveHandling();
  }

  initResponsiveHandling() {
    const checkMobile = () => {
      const isMobile = window.innerWidth < 768;
      if (isMobile !== this.state.isMobile) {
        this.dispatch({ type: 'SET_MOBILE', payload: isMobile });
      }
    };

    window.addEventListener('resize', checkMobile);
    checkMobile(); // Initial check
  }

  reducer(state, action) {
    switch (action.type) {
      case 'SET_PAGE':
        return { ...state, currentPage: action.payload };

      case 'SET_LOADING':
        return { ...state, loading: action.payload };

      case 'SET_MOBILE':
        return { ...state, isMobile: action.payload };

      case 'SHOW_MODAL':
        return {
          ...state,
          modal: {
            isOpen: true,
            type: action.payload.type,
            data: action.payload.data
          }
        };

      case 'HIDE_MODAL':
        return {
          ...state,
          modal: {
            isOpen: false,
            type: null,
            data: null
          }
        };

      case 'ADD_NOTIFICATION':
        return {
          ...state,
          notifications: [...state.notifications, action.payload]
        };

      case 'REMOVE_NOTIFICATION':
        return {
          ...state,
          notifications: state.notifications.filter(n => n.id !== action.payload)
        };

      default:
        return state;
    }
  }
}

// Create global store instance
const appStore = new AppStore();
```

## Performance Optimizations

### Component Memoization
```javascript
// Memoization utility
function memo(ComponentClass) {
  let prevProps = null;
  let prevResult = null;

  return class MemoizedComponent extends ComponentClass {
    render() {
      // Simple props comparison (deep comparison would be more robust)
      if (JSON.stringify(this.props) === JSON.stringify(prevProps) && prevResult) {
        return prevResult;
      }

      prevProps = { ...this.props };
      prevResult = super.render();
      return prevResult;
    }
  };
}

// Usage
const MemoizedHomePage = memo(HomePage);
```

### Virtual Scrolling for Lists
```javascript
class VirtualList extends Component {
  constructor(props = {}) {
    super(props);
    this.items = props.items || [];
    this.itemHeight = props.itemHeight || 50;
    this.containerHeight = props.containerHeight || 400;
    this.scrollTop = 0;
  }

  render() {
    const totalHeight = this.items.length * this.itemHeight;
    const visibleItems = this.getVisibleItems();

    return `
      <div class="virtual-list" style="height: ${this.containerHeight}px; overflow: auto;">
        <div style="height: ${totalHeight}px; position: relative;">
          ${visibleItems.map(item => `
            <div style="position: absolute; top: ${item.top}px; width: 100%;">
              ${this.renderItem(item.data)}
            </div>
          `).join('')}
        </div>
      </div>
    `;
  }

  getVisibleItems() {
    const startIndex = Math.floor(this.scrollTop / this.itemHeight);
    const endIndex = Math.min(
      startIndex + Math.ceil(this.containerHeight / this.itemHeight),
      this.items.length - 1
    );

    const visibleItems = [];
    for (let i = startIndex; i <= endIndex; i++) {
      visibleItems.push({
        data: this.items[i],
        top: i * this.itemHeight
      });
    }

    return visibleItems;
  }

  renderItem(item) {
    // To be implemented by subclasses
    return `<div>${item.name}</div>`;
  }

  componentDidMount() {
    const container = this.element.querySelector('.virtual-list');
    container.addEventListener('scroll', (e) => {
      this.scrollTop = e.target.scrollTop;
      this.render();
    });
  }
}
```

### Lazy Loading
```javascript
// Lazy loading utility
function lazyLoad(ComponentClass) {
  let component = null;

  return class LazyComponent extends Component {
    constructor(props = {}) {
      super(props);
      this.loaded = false;
    }

    async render() {
      if (!this.loaded) {
        // Show loading state
        this.loaded = true;

        // Dynamically import component
        try {
          const module = await import(`./${ComponentClass.name}.js`);
          component = new module.default(this.props);
          return component.render();
        } catch (error) {
          return '<div class="text-red-500">Failed to load component</div>';
        }
      }

      return component.render();
    }
  };
}

// Usage
const LazyTournamentPage = lazyLoad(TournamentPage);
```

## Testing Strategy

### Unit Tests
```javascript
// Component testing
describe('Button Component', () => {
  let button;

  beforeEach(() => {
    button = new Button({
      text: 'Click me',
      variant: 'primary',
      onClick: jest.fn()
    });
  });

  test('renders correctly', () => {
    const html = button.render();
    expect(html).toContain('Click me');
    expect(html).toContain('bg-orange-500');
  });

  test('handles click events', () => {
    // Mock DOM event
    const mockEvent = { preventDefault: jest.fn() };
    button.props.onClick(mockEvent);
    expect(button.props.onClick).toHaveBeenCalledWith(mockEvent);
  });
});

// Store testing
describe('UserStore', () => {
  let store;

  beforeEach(() => {
    store = new UserStore();
  });

  test('handles login success', () => {
    const user = { id: 1, name: 'Test User' };
    store.dispatch({
      type: 'USER_LOGIN_SUCCESS',
      payload: { user }
    });

    const state = store.getState();
    expect(state.isAuthenticated).toBe(true);
    expect(state.currentUser).toEqual(user);
  });
});
```

### Integration Tests
```javascript
// API integration testing
describe('User API Integration', () => {
  test('fetches user profile', async () => {
    const userService = new UserService();
    const response = await userService.getProfile();

    expect(response.success).toBe(true);
    expect(response.data.user).toHaveProperty('id');
    expect(response.data.user).toHaveProperty('name');
  });
});
```

## Build and Deployment

### Vite Configuration
```javascript
// vite.config.js
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  root: 'src/Frontend',
  build: {
    outDir: '../../public/assets',
    rollupOptions: {
      input: {
        main: resolve(__dirname, 'src/Frontend/index.js'),
        styles: resolve(__dirname, 'src/Frontend/styles/main.css')
      }
    }
  },
  server: {
    proxy: {
      '/api': 'http://localhost:8000'
    }
  }
});
```

### Development Workflow
```bash
# Development
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview

# Run tests
npm test

# Run linting
npm run lint
```

This frontend architecture provides a solid foundation for building a scalable, maintainable single-page application with excellent user experience and developer productivity.