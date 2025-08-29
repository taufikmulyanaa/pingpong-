# PingPong+ API Architecture

## Overview
RESTful API architecture for the PingPong+ platform with proper separation of concerns, security, and scalability.

## API Design Principles

### RESTful Design
- Resource-based URLs
- HTTP methods for CRUD operations
- Proper HTTP status codes
- JSON responses
- Versioned endpoints

### Security
- JWT authentication
- Rate limiting
- Input validation
- CORS handling
- SQL injection prevention

### Performance
- Response caching
- Database query optimization
- Pagination for large datasets
- Compression
- CDN integration

## API Endpoints Structure

### Base URL
```
https://api.pingpongplus.com/v1/
```

### Authentication Endpoints

#### POST /auth/login
User login endpoint.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "user": {
      "id": 1,
      "name": "Ahmad Supardi",
      "email": "user@example.com",
      "photo": "https://...",
      "elo": 1720,
      "skill": "Menengah"
    },
    "token": "jwt_token_here",
    "expires_at": "2025-08-30T03:21:36.000Z"
  }
}
```

#### POST /auth/register
User registration endpoint.

**Request:**
```json
{
  "name": "Ahmad Supardi",
  "email": "user@example.com",
  "password": "password123",
  "age": 28,
  "location": "Jakarta",
  "skill": "Menengah"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Registration successful",
  "data": {
    "user": {
      "id": 1,
      "name": "Ahmad Supardi",
      "email": "user@example.com"
    }
  }
}
```

#### POST /auth/logout
User logout endpoint.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Response:**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

#### POST /auth/refresh
Refresh JWT token.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Response:**
```json
{
  "success": true,
  "data": {
    "token": "new_jwt_token_here",
    "expires_at": "2025-08-30T03:21:36.000Z"
  }
}
```

### User Management Endpoints

#### GET /users/profile
Get current user profile.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Response:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "name": "Ahmad Supardi",
      "email": "user@example.com",
      "age": 28,
      "location": "Jakarta",
      "elo": 1720,
      "skill": "Menengah",
      "style": "All-round",
      "photo": "https://...",
      "club": {
        "id": 1,
        "name": "Jakarta PP Club",
        "role": "player"
      },
      "stats": {
        "matches": 47,
        "wins": 32,
        "win_rate": 68,
        "tournaments": 5,
        "rank": 2
      },
      "equipment": {
        "blade": "Butterfly Viscaria",
        "rubber_fh": "Dignics 05",
        "rubber_bh": "Dignics 05"
      }
    }
  }
}
```

#### PUT /users/profile
Update user profile.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Request:**
```json
{
  "name": "Ahmad Supardi",
  "location": "Jakarta Selatan",
  "skill": "Mahir",
  "equipment": {
    "blade": "Butterfly Viscaria",
    "rubber_fh": "Dignics 05",
    "rubber_bh": "Dignics 05"
  }
}
```

#### GET /users/search
Search users with filters.

**Query Parameters:**
- `q`: Search query (name, location)
- `skill`: Skill level filter
- `online`: Online status filter
- `location`: Location filter
- `limit`: Results limit (default: 20)
- `offset`: Pagination offset

**Response:**
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": 2,
        "name": "Budi Rahman",
        "photo": "https://...",
        "elo": 1650,
        "skill": "Menengah",
        "location": "Tebet, Jakarta",
        "distance": 2.5,
        "online": true,
        "last_active": "5 menit lalu"
      }
    ],
    "pagination": {
      "total": 45,
      "limit": 20,
      "offset": 0,
      "has_more": true
    }
  }
}
```

#### GET /users/{id}
Get specific user profile.

**Response:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 2,
      "name": "Budi Rahman",
      "age": 32,
      "location": "Tebet, Jakarta",
      "elo": 1650,
      "skill": "Menengah",
      "style": "Bertahan",
      "photo": "https://...",
      "stats": {
        "matches": 62,
        "wins": 38,
        "win_rate": 61,
        "tournaments": 5
      },
      "rating": 4.5,
      "reviews": 10
    }
  }
}
```

### Tournament Endpoints

#### GET /tournaments
List tournaments with filters.

**Query Parameters:**
- `status`: open, upcoming, completed
- `category`: Singles • Menengah, Doubles • Pemula
- `location`: City or venue
- `upcoming`: true/false
- `limit`: Results limit
- `offset`: Pagination offset

**Response:**
```json
{
  "success": true,
  "data": {
    "tournaments": [
      {
        "id": 1,
        "name": "Jakarta Weekend Cup",
        "date": "2025-09-07",
        "location": "GOR Senayan",
        "category": "Singles • Menengah",
        "fee": 75000,
        "participants": 28,
        "max_participants": 32,
        "status": "Open",
        "prize": 5000000,
        "organizer": "Jakarta PP Club"
      }
    ],
    "pagination": {
      "total": 15,
      "limit": 10,
      "offset": 0,
      "has_more": true
    }
  }
}
```

#### POST /tournaments
Create new tournament.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Request:**
```json
{
  "name": "Jakarta Weekend Cup",
  "description": "Weekly tournament for intermediate players",
  "date": "2025-09-07",
  "location": "GOR Senayan",
  "category": "Singles • Menengah",
  "max_participants": 32,
  "fee": 75000,
  "prize": 5000000,
  "rules": "Best of 5 games, 11 point system",
  "schedule": [
    {
      "round": "Round 1",
      "time": "08:00-12:00"
    }
  ]
}
```

#### GET /tournaments/{id}
Get tournament details.

**Response:**
```json
{
  "success": true,
  "data": {
    "tournament": {
      "id": 1,
      "name": "Jakarta Weekend Cup",
      "description": "Weekly tournament for intermediate players",
      "date": "2025-09-07",
      "location": "GOR Senayan",
      "category": "Singles • Menengah",
      "max_participants": 32,
      "current_participants": 28,
      "fee": 75000,
      "prize": 5000000,
      "status": "Open",
      "organizer": {
        "id": 1,
        "name": "Jakarta PP Club"
      },
      "participants": [
        {
          "id": 1,
          "name": "Ahmad Supardi",
          "elo": 1720
        }
      ],
      "schedule": [
        {
          "round": "Round 1",
          "time": "08:00-12:00"
        }
      ]
    }
  }
}
```

#### POST /tournaments/{id}/register
Register for tournament.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Response:**
```json
{
  "success": true,
  "message": "Successfully registered for tournament",
  "data": {
    "registration_id": 123,
    "status": "registered"
  }
}
```

### Club Endpoints

#### GET /clubs
List clubs with filters.

**Query Parameters:**
- `city`: City filter
- `province`: Province filter
- `verified`: true/false
- `limit`: Results limit
- `offset`: Pagination offset

**Response:**
```json
{
  "success": true,
  "data": {
    "clubs": [
      {
        "id": 1,
        "name": "Jakarta PP Club",
        "short_name": "JPC",
        "city": "Jakarta",
        "province": "DKI Jakarta",
        "logo": "https://...",
        "members": 156,
        "rating": 4.8,
        "verified": true,
        "team_ranking": 2
      }
    ],
    "pagination": {
      "total": 25,
      "limit": 10,
      "offset": 0,
      "has_more": true
    }
  }
}
```

#### GET /clubs/{id}
Get club details.

**Response:**
```json
{
  "success": true,
  "data": {
    "club": {
      "id": 1,
      "name": "Jakarta PP Club",
      "description": "Leading table tennis club in Jakarta",
      "city": "Jakarta",
      "address": "Jl. Sudirman No. 123",
      "logo": "https://...",
      "photos": ["https://...", "https://..."],
      "facilities": ["12 tournament tables", "AC", "Parking"],
      "members": 156,
      "coaches": 8,
      "rating": 4.8,
      "membership": {
        "monthly": 200000,
        "yearly": 2000000
      },
      "contact": {
        "phone": "021-5551234",
        "email": "info@jakartapc.com"
      }
    }
  }
}
```

#### POST /clubs/{id}/join
Join a club.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Response:**
```json
{
  "success": true,
  "message": "Successfully joined club",
  "data": {
    "membership_id": 456,
    "role": "player",
    "status": "active"
  }
}
```

### Venue Endpoints

#### GET /venues
List venues with filters.

**Query Parameters:**
- `city`: City filter
- `type`: Indoor, Outdoor
- `rating`: Minimum rating
- `limit`: Results limit
- `offset`: Pagination offset

**Response:**
```json
{
  "success": true,
  "data": {
    "venues": [
      {
        "id": 1,
        "name": "GOR Senayan",
        "address": "Jl. Pintu Satu Senayan",
        "city": "Jakarta",
        "rating": 4.5,
        "reviews": 125,
        "tables": 12,
        "type": "Indoor + AC",
        "price": 35000,
        "facilities": ["AC", "Parking", "Canteen"]
      }
    ],
    "pagination": {
      "total": 50,
      "limit": 10,
      "offset": 0,
      "has_more": true
    }
  }
}
```

#### GET /venues/{id}
Get venue details.

**Response:**
```json
{
  "success": true,
  "data": {
    "venue": {
      "id": 1,
      "name": "GOR Senayan",
      "description": "Premier table tennis venue in Jakarta",
      "address": "Jl. Pintu Satu Senayan, Jakarta Pusat",
      "phone": "021-5551234",
      "rating": 4.5,
      "reviews": 125,
      "tables": 12,
      "facilities": ["AC", "Parking", "Canteen", "Shower"],
      "operating_hours": "06:00-22:00",
      "photos": ["https://...", "https://..."],
      "reviews_list": [
        {
          "id": 1,
          "user": "Ahmad Supardi",
          "rating": 5,
          "comment": "Excellent facilities and clean",
          "created_at": "2025-08-25T10:30:00Z"
        }
      ]
    }
  }
}
```

#### POST /venues/{id}/review
Add venue review.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Request:**
```json
{
  "rating": 5,
  "cleanliness_rating": 5,
  "equipment_rating": 4,
  "staff_rating": 5,
  "comment": "Excellent facilities and clean",
  "visit_date": "2025-08-25"
}
```

### Marketplace Endpoints

#### GET /marketplace
List marketplace items.

**Query Parameters:**
- `category`: Bats, Balls, Shoes, etc.
- `condition`: Baru, Baik, etc.
- `price_min`: Minimum price
- `price_max`: Maximum price
- `location`: Location filter
- `limit`: Results limit
- `offset`: Pagination offset

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": 1,
        "name": "Bat Butterfly Timo Boll ALC",
        "price": 450000,
        "condition": "Baik",
        "location": "Jaksel, 2.1km",
        "photo": "https://...",
        "seller": {
          "id": 2,
          "name": "Budi Rahman",
          "rating": 4.5
        },
        "posted_date": "2025-08-25"
      }
    ],
    "pagination": {
      "total": 30,
      "limit": 10,
      "offset": 0,
      "has_more": true
    }
  }
}
```

#### POST /marketplace
Create marketplace listing.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Request:**
```json
{
  "name": "Bat Butterfly Timo Boll ALC",
  "description": "Used for 6 months, still in great condition",
  "category": "Bat",
  "brand": "Butterfly",
  "condition": "Baik",
  "price": 450000,
  "location": "Jakarta Selatan",
  "images": ["https://...", "https://..."],
  "description_details": {
    "brand": "Butterfly",
    "model": "Timo Boll ALC",
    "condition": "Used for 6 months, still in great condition",
    "meeting_point": "COD di Blok M atau Senayan"
  }
}
```

#### GET /marketplace/{id}
Get marketplace item details.

**Response:**
```json
{
  "success": true,
  "data": {
    "item": {
      "id": 1,
      "name": "Bat Butterfly Timo Boll ALC",
      "price": 450000,
      "condition": "Baik",
      "location": "Jaksel, 2.1km",
      "images": ["https://...", "https://..."],
      "description": {
        "brand": "Butterfly",
        "model": "Timo Boll ALC",
        "condition": "Used for 6 months, still in great condition",
        "meeting_point": "COD di Blok M atau Senayan"
      },
      "seller": {
        "id": 2,
        "name": "Budi Rahman",
        "photo": "https://...",
        "rating": 4.5,
        "reviews": 10
      },
      "posted_date": "2025-08-25",
      "status": "Tersedia"
    }
  }
}
```

### Feed Endpoints

#### GET /feed
Get community feed posts.

**Query Parameters:**
- `limit`: Posts limit (default: 20)
- `offset`: Pagination offset
- `type`: Filter by post type

**Response:**
```json
{
  "success": true,
  "data": {
    "posts": [
      {
        "id": 1,
        "type": "match_report",
        "user": {
          "id": 2,
          "name": "Budi Rahman",
          "photo": "https://..."
        },
        "content": "Menang 3-1 vs Andi! Seru banget match hari ini 🔥",
        "image": "https://...",
        "score": "3-1",
        "opponent": "Andi Saputra",
        "venue": "GOR Senayan",
        "likes": 12,
        "comments": 5,
        "created_at": "2025-08-28T14:30:00Z"
      }
    ],
    "pagination": {
      "total": 100,
      "limit": 20,
      "offset": 0,
      "has_more": true
    }
  }
}
```

#### POST /feed
Create new feed post.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Request:**
```json
{
  "type": "match_report",
  "content": "Menang 3-1 vs Andi! Seru banget match hari ini 🔥",
  "image": "https://...",
  "score": "3-1",
  "opponent": "Andi Saputra",
  "venue": "GOR Senayan"
}
```

#### POST /feed/{id}/like
Like a feed post.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Request:**
```json
{
  "reaction_type": "like"
}
```

#### POST /feed/{id}/comment
Comment on a feed post.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Request:**
```json
{
  "content": "Selamat ya! Bagus permainannya!",
  "parent_id": null  // For nested comments
}
```

### Notification Endpoints

#### GET /notifications
Get user notifications.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

**Query Parameters:**
- `limit`: Notifications limit
- `offset`: Pagination offset
- `unread_only`: true/false

**Response:**
```json
{
  "success": true,
  "data": {
    "notifications": [
      {
        "id": 1,
        "type": "match_request",
        "title": "Match Request",
        "message": "Budi Rahman wants to play with you",
        "data": {
          "from_user_id": 2,
          "match_type": "singles"
        },
        "is_read": false,
        "created_at": "2025-08-28T15:30:00Z"
      }
    ],
    "pagination": {
      "total": 25,
      "limit": 10,
      "offset": 0,
      "has_more": true
    }
  }
}
```

#### PUT /notifications/{id}/read
Mark notification as read.

**Headers:**
```
Authorization: Bearer jwt_token_here
```

## Error Handling

### Standard Error Response Format
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Input validation failed",
    "details": {
      "field": "email",
      "reason": "Invalid email format"
    }
  }
}
```

### Common HTTP Status Codes
- `200`: Success
- `201`: Created
- `400`: Bad Request (validation errors)
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Not Found
- `409`: Conflict (duplicate data)
- `422`: Unprocessable Entity
- `429`: Too Many Requests (rate limited)
- `500`: Internal Server Error

## Rate Limiting

### Rate Limits by Endpoint Type
- **Authentication**: 5 requests per minute per IP
- **User Profile**: 60 requests per hour per user
- **Search/List**: 100 requests per hour per user
- **Create/Update**: 30 requests per hour per user
- **File Upload**: 10 requests per hour per user

### Rate Limit Headers
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1638360000
```

## Pagination

### Standard Pagination Format
```json
{
  "success": true,
  "data": [...],
  "pagination": {
    "total": 150,
    "limit": 20,
    "offset": 0,
    "has_more": true,
    "next_offset": 20,
    "prev_offset": null
  }
}
```

## Caching Strategy

### Response Caching
- Public data (venues, clubs): Cache for 15 minutes
- User-specific data: Cache for 5 minutes
- Real-time data (notifications): No cache

### Cache Headers
```
Cache-Control: public, max-age=900
ETag: "abc123"
Last-Modified: Wed, 29 Aug 2025 03:21:36 GMT
```

## Versioning Strategy

### API Versioning
- URL-based versioning: `/v1/`, `/v2/`
- Backward compatibility maintained for 2 versions
- Deprecation warnings in response headers
- Migration guides provided

### Breaking Changes
- Major version bump for breaking changes
- Feature flags for gradual rollout
- Comprehensive testing before release