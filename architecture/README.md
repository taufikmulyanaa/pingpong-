# PingPong+ - Table Tennis Social Platform

## Overview

PingPong+ is a comprehensive social platform designed specifically for the table tennis community in Indonesia. The platform connects players of all skill levels, facilitates tournament organization, provides venue discovery, and enables equipment trading through a dedicated marketplace.

## 🎯 Mission

To become the go-to platform for table tennis enthusiasts by providing a complete ecosystem that supports skill development, community building, and competitive play.

## ✨ Key Features

### 👥 Player Connection
- **Profile Management**: Comprehensive player profiles with skill ratings, playing style, and statistics
- **Smart Matching**: Find players based on skill level, location, and availability
- **Social Features**: Friend system, messaging, and community feed
- **ELO Rating System**: Automated skill rating with seasonal resets

### 🏆 Tournament Management
- **Easy Creation**: Simple tournament setup with customizable rules
- **Bracket Generation**: Automated tournament brackets and scheduling
- **Live Scoring**: Real-time score tracking and result management
- **Prize Management**: Support for cash prizes and certificates

### 🏢 Venue Discovery
- **Comprehensive Directory**: Complete venue database with ratings and reviews
- **Facility Details**: Equipment availability, pricing, and amenities
- **Check-in System**: Real-time activity tracking
- **Booking Integration**: Direct venue booking capabilities

### 🛒 Marketplace
- **Equipment Trading**: Buy and sell table tennis equipment
- **COD Support**: Safe cash-on-delivery transactions
- **Quality Assurance**: Item verification and seller ratings
- **Price Guidance**: Market price recommendations

## 🏗️ Architecture

### Technology Stack

#### Backend
- **Language**: PHP 8.1+
- **Database**: MySQL 8.0
- **Authentication**: JWT + Session-based
- **Caching**: Redis
- **File Storage**: Local filesystem with CDN integration

#### Frontend
- **Framework**: Vanilla JavaScript (ES6+)
- **Styling**: Tailwind CSS
- **Build Tool**: Vite
- **Components**: Web Components (Custom Elements)

#### DevOps
- **Containerization**: Docker
- **CI/CD**: GitHub Actions
- **Monitoring**: Application logs + error tracking
- **Deployment**: Automated with rollback capability

### System Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Browser   │    │   API Gateway   │    │   Load Balancer │
│                 │    │                 │    │                 │
│ - React/Vue App │◄──►│ - Rate Limiting │◄──►│ - Nginx         │
│ - PWA Features  │    │ - Auth Check    │    │ - SSL           │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   CDN (Assets)  │    │   Application   │    │   Database      │
│                 │    │   Servers       │    │                 │
│ - Static Files  │    │                 │    │ - MySQL 8.0     │
│ - Image Cache   │    │ - PHP 8.1       │    │ - Redis Cache   │
└─────────────────┘    │ - Node.js       │    └─────────────────┘
                       │ - Queue Workers │
                       └─────────────────┘
```

## 📁 Project Structure

```
pingpong+/
├── 📁 public/                 # Web root
│   ├── index.php             # Main entry point
│   ├── assets/               # Compiled assets
│   └── uploads/              # User uploads
├── 📁 src/                   # Source code
│   ├── 📁 Backend/           # PHP backend
│   │   ├── 📁 Controllers/   # API controllers
│   │   ├── 📁 Models/        # Database models
│   │   ├── 📁 Services/      # Business logic
│   │   └── 📁 Config/        # Configuration
│   └── 📁 Frontend/          # JavaScript frontend
│       ├── 📁 components/    # UI components
│       ├── 📁 pages/         # Page components
│       ├── 📁 services/      # API services
│       └── 📁 stores/        # State management
├── 📁 tests/                 # Test suites
├── 📁 docs/                  # Documentation
├── 📁 scripts/               # Build scripts
└── 📁 config/                # Environment configs
```

## 🚀 Quick Start

### Prerequisites
- PHP 8.1 or higher
- MySQL 8.0 or higher
- Node.js 18 or higher
- Composer (PHP package manager)
- npm or yarn

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/your-org/pingpong-plus.git
   cd pingpong-plus
   ```

2. **Setup backend**
   ```bash
   cd src/Backend
   composer install
   cp .env.example .env
   # Configure database settings in .env
   ```

3. **Setup frontend**
   ```bash
   cd ../Frontend
   npm install
   cp .env.example .env
   # Configure API endpoints in .env
   ```

4. **Setup database**
   ```bash
   cd ../Database
   # Run schema.sql and seed data
   ```

5. **Start development servers**
   ```bash
   # Backend (from src/Backend)
   php -S localhost:8000

   # Frontend (from src/Frontend)
   npm run dev
   ```

6. **Access the application**
   - Frontend: http://localhost:3000
   - API: http://localhost:8000/api/v1

## 🔐 Security Features

### Authentication
- JWT token-based authentication
- Secure password hashing (Argon2ID)
- Session management with automatic expiration
- Multi-factor authentication support

### Authorization
- Role-based access control (RBAC)
- User tier system (Basic, Premium, Club Admin)
- Resource-level permissions
- API rate limiting

### Data Protection
- Input validation and sanitization
- SQL injection prevention
- XSS protection
- CSRF protection
- File upload security

## 📊 User Tiers

### Basic (Free)
- Player profiles and matching
- Community feed access
- Venue directory browsing
- Basic marketplace access (buying only)

### Premium Player (Rp30,000/month)
- All Basic features
- Tournament creation (up to 10/month)
- Advanced search and filtering
- Marketplace selling capabilities
- Priority customer support

### Club Administrator (Rp150,000/month)
- All Premium features
- Unlimited tournament creation
- Club management tools
- Advanced analytics and reporting
- Bulk operations and API access

## 🧪 Testing

### Backend Testing
```bash
cd src/Backend
composer test
```

### Frontend Testing
```bash
cd src/Frontend
npm test
```

### End-to-End Testing
```bash
npm run test:e2e
```

## 🚀 Deployment

### Development
```bash
npm run build:dev
```

### Production
```bash
npm run build
npm run deploy
```

### Docker
```bash
docker-compose up -d
```

## 📈 Performance Optimization

### Frontend
- Code splitting and lazy loading
- Asset optimization and compression
- Progressive Web App features
- Service worker caching

### Backend
- Database query optimization
- Redis caching layers
- API response compression
- Background job processing

### Infrastructure
- CDN for static assets
- Database read replicas
- Load balancing
- Horizontal scaling support

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Development Guidelines
- Follow PSR-12 coding standards for PHP
- Use ESLint and Prettier for JavaScript
- Write comprehensive tests for new features
- Update documentation for API changes
- Ensure all tests pass before submitting PR

## 📝 API Documentation

### Authentication Endpoints
- `POST /api/v1/auth/login` - User login
- `POST /api/v1/auth/register` - User registration
- `POST /api/v1/auth/logout` - User logout
- `POST /api/v1/auth/refresh` - Refresh JWT token

### User Management
- `GET /api/v1/users/profile` - Get user profile
- `PUT /api/v1/users/profile` - Update user profile
- `GET /api/v1/users/search` - Search users

### Tournament Management
- `GET /api/v1/tournaments` - List tournaments
- `POST /api/v1/tournaments` - Create tournament
- `GET /api/v1/tournaments/{id}` - Get tournament details
- `POST /api/v1/tournaments/{id}/register` - Register for tournament

### Club Management
- `GET /api/v1/clubs` - List clubs
- `POST /api/v1/clubs` - Create club
- `GET /api/v1/clubs/{id}` - Get club details
- `POST /api/v1/clubs/{id}/join` - Join club

### Marketplace
- `GET /api/v1/marketplace` - List marketplace items
- `POST /api/v1/marketplace` - Create marketplace listing
- `GET /api/v1/marketplace/{id}` - Get item details

### Feed & Social
- `GET /api/v1/feed` - Get community feed
- `POST /api/v1/feed` - Create feed post
- `POST /api/v1/feed/{id}/like` - Like a post
- `POST /api/v1/feed/{id}/comment` - Comment on a post

## 📞 Support

### Community Support
- **Forum**: [Community Forum](https://forum.pingpongplus.com)
- **Discord**: [Join our Discord](https://discord.gg/pingpongplus)
- **Documentation**: [Developer Docs](https://docs.pingpongplus.com)

### Business Support
- **Email**: support@pingpongplus.com
- **Phone**: +62 21 555-0123
- **Business Hours**: Mon-Fri 9:00-17:00 WIB

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- Table tennis community in Indonesia
- Open source contributors
- Beta testers and early adopters

---

**Built with ❤️ for the Indonesian table tennis community**

*PingPong+ - Where Players Connect, Compete, and Grow* 🏓