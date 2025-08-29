# PingPong+ Database Schema

## Overview
Complete database schema for the PingPong+ table tennis social platform.

## Core Tables

### 1. users
```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    age INT,
    location VARCHAR(255),
    elo_rating INT DEFAULT 1500,
    skill_level ENUM('Pemula', 'Menengah', 'Mahir', 'Expert') DEFAULT 'Pemula',
    playing_style ENUM('Menyerang', 'Bertahan', 'All-round') DEFAULT 'All-round',
    availability JSON, -- Store as JSON: ["Pagi", "Sore", "Malam"]
    photo VARCHAR(500),
    bio TEXT,
    phone VARCHAR(20),
    is_online BOOLEAN DEFAULT FALSE,
    last_seen TIMESTAMP NULL,
    email_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_email (email),
    INDEX idx_location (location),
    INDEX idx_skill_level (skill_level),
    INDEX idx_elo_rating (elo_rating),
    INDEX idx_is_online (is_online)
);
```

### 2. user_sessions
```sql
CREATE TABLE user_sessions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    session_token VARCHAR(255) UNIQUE NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id),
    INDEX idx_session_token (session_token),
    INDEX idx_expires_at (expires_at)
);
```

### 3. clubs
```sql
CREATE TABLE clubs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    short_name VARCHAR(50),
    city VARCHAR(100) NOT NULL,
    province VARCHAR(100) NOT NULL,
    address TEXT NOT NULL,
    established YEAR,
    description TEXT,
    logo VARCHAR(500),
    photos JSON, -- Store as JSON array of image URLs
    website VARCHAR(255),
    phone VARCHAR(20),
    email VARCHAR(255),
    monthly_fee DECIMAL(10,2),
    yearly_fee DECIMAL(10,2),
    student_fee DECIMAL(10,2),
    youth_fee DECIMAL(10,2),
    facilities JSON, -- Store as JSON array: ["AC", "Parking", "Canteen"]
    member_count INT DEFAULT 0,
    active_members INT DEFAULT 0,
    coach_count INT DEFAULT 0,
    rating DECIMAL(3,2) DEFAULT 0.00,
    review_count INT DEFAULT 0,
    verified BOOLEAN DEFAULT FALSE,
    team_ranking INT DEFAULT 0,
    recent_matches INT DEFAULT 0,
    win_rate DECIMAL(5,2) DEFAULT 0.00,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_city (city),
    INDEX idx_province (province),
    INDEX idx_verified (verified),
    INDEX idx_team_ranking (team_ranking)
);
```

### 4. club_members
```sql
CREATE TABLE club_members (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    club_id INT NOT NULL,
    role ENUM('player', 'coach', 'admin') DEFAULT 'player',
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('active', 'inactive', 'suspended') DEFAULT 'active',

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (club_id) REFERENCES clubs(id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_club (user_id, club_id),
    INDEX idx_user_id (user_id),
    INDEX idx_club_id (club_id),
    INDEX idx_role (role),
    INDEX idx_status (status)
);
```

### 5. tournaments
```sql
CREATE TABLE tournaments (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    tournament_date DATE NOT NULL,
    start_time TIME,
    end_time TIME,
    location VARCHAR(255) NOT NULL,
    venue_details TEXT,
    category VARCHAR(100) NOT NULL, -- "Singles • Menengah", "Doubles • Pemula"
    tournament_type ENUM('individual', 'club', 'open') DEFAULT 'individual',
    format VARCHAR(100), -- "Single Elimination", "Round Robin"
    max_participants INT NOT NULL,
    current_participants INT DEFAULT 0,
    entry_fee DECIMAL(10,2) DEFAULT 0,
    prize_money DECIMAL(10,2) DEFAULT 0,
    registration_deadline DATE,
    status ENUM('draft', 'open', 'closed', 'in_progress', 'completed', 'cancelled') DEFAULT 'draft',
    organizer_id INT, -- User who created the tournament
    club_id INT NULL, -- Club organizing the tournament
    rules TEXT,
    schedule JSON, -- Store tournament schedule as JSON
    contact_info JSON, -- Contact details as JSON
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (organizer_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (club_id) REFERENCES clubs(id) ON DELETE SET NULL,
    INDEX idx_tournament_date (tournament_date),
    INDEX idx_status (status),
    INDEX idx_category (category),
    INDEX idx_organizer_id (organizer_id),
    INDEX idx_club_id (club_id)
);
```

### 6. tournament_participants
```sql
CREATE TABLE tournament_participants (
    id INT PRIMARY KEY AUTO_INCREMENT,
    tournament_id INT NOT NULL,
    user_id INT NOT NULL,
    registration_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('registered', 'confirmed', 'withdrawn', 'disqualified') DEFAULT 'registered',
    seed_number INT NULL,
    payment_status ENUM('pending', 'paid', 'refunded') DEFAULT 'pending',
    payment_amount DECIMAL(10,2),
    payment_date TIMESTAMP NULL,

    FOREIGN KEY (tournament_id) REFERENCES tournaments(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_tournament_user (tournament_id, user_id),
    INDEX idx_tournament_id (tournament_id),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_payment_status (payment_status)
);
```

### 7. tournament_matches
```sql
CREATE TABLE tournament_matches (
    id INT PRIMARY KEY AUTO_INCREMENT,
    tournament_id INT NOT NULL,
    round_number INT NOT NULL,
    match_number INT NOT NULL,
    player1_id INT,
    player2_id INT,
    winner_id INT NULL,
    score_player1 INT DEFAULT 0,
    score_player2 INT DEFAULT 0,
    detailed_score JSON, -- Store set-by-set scores as JSON
    match_date DATE,
    start_time TIME,
    end_time TIME,
    venue VARCHAR(255),
    status ENUM('scheduled', 'in_progress', 'completed', 'cancelled') DEFAULT 'scheduled',
    referee_id INT NULL,
    notes TEXT,

    FOREIGN KEY (tournament_id) REFERENCES tournaments(id) ON DELETE CASCADE,
    FOREIGN KEY (player1_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (player2_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (winner_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (referee_id) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_tournament_id (tournament_id),
    INDEX idx_round_number (round_number),
    INDEX idx_player1_id (player1_id),
    INDEX idx_player2_id (player2_id),
    INDEX idx_winner_id (winner_id),
    INDEX idx_status (status),
    INDEX idx_match_date (match_date)
);
```

### 8. venues
```sql
CREATE TABLE venues (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    address TEXT NOT NULL,
    city VARCHAR(100) NOT NULL,
    province VARCHAR(100) NOT NULL,
    coordinates POINT NULL, -- For spatial queries
    phone VARCHAR(20),
    email VARCHAR(255),
    website VARCHAR(255),
    description TEXT,
    photo VARCHAR(500),
    photos JSON, -- Additional photos as JSON array
    type VARCHAR(100), -- "Indoor + AC", "Outdoor", "Indoor"
    tables_count INT NOT NULL,
    facilities JSON, -- Facilities as JSON array
    operating_hours JSON, -- Hours as JSON: {"monday": "06:00-22:00"}
    price_per_hour DECIMAL(8,2),
    price_unit VARCHAR(50) DEFAULT 'per jam',
    rating DECIMAL(3,2) DEFAULT 0.00,
    review_count INT DEFAULT 0,
    verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_city (city),
    INDEX idx_province (province),
    INDEX idx_type (type),
    INDEX idx_verified (verified),
    INDEX idx_rating (rating),
    SPATIAL INDEX idx_coordinates (coordinates)
);
```

### 9. venue_reviews
```sql
CREATE TABLE venue_reviews (
    id INT PRIMARY KEY AUTO_INCREMENT,
    venue_id INT NOT NULL,
    user_id INT NOT NULL,
    rating INT NOT NULL CHECK (rating >= 1 AND rating <= 5),
    cleanliness_rating INT CHECK (cleanliness_rating >= 1 AND cleanliness_rating <= 5),
    equipment_rating INT CHECK (equipment_rating >= 1 AND equipment_rating <= 5),
    staff_rating INT CHECK (staff_rating >= 1 AND staff_rating <= 5),
    comment TEXT,
    photos JSON, -- Review photos as JSON array
    visit_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (venue_id) REFERENCES venues(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_venue_user (venue_id, user_id),
    INDEX idx_venue_id (venue_id),
    INDEX idx_user_id (user_id),
    INDEX idx_rating (rating)
);
```

### 10. marketplace_items
```sql
CREATE TABLE marketplace_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    seller_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100) NOT NULL,
    brand VARCHAR(100),
    condition_status VARCHAR(50) NOT NULL, -- "Baru", "Seperti Baru", "Baik", etc.
    price DECIMAL(12,2) NOT NULL,
    location VARCHAR(255) NOT NULL,
    photo VARCHAR(500),
    images JSON, -- Additional images as JSON array
    status ENUM('Tersedia', 'Terjual', 'Dihapus') DEFAULT 'Tersedia',
    posted_date DATE DEFAULT (CURRENT_DATE),
    view_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (seller_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_seller_id (seller_id),
    INDEX idx_category (category),
    INDEX idx_brand (brand),
    INDEX idx_condition_status (condition_status),
    INDEX idx_price (price),
    INDEX idx_location (location),
    INDEX idx_status (status),
    INDEX idx_posted_date (posted_date)
);
```

### 11. marketplace_transactions
```sql
CREATE TABLE marketplace_transactions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    item_id INT NOT NULL,
    buyer_id INT NOT NULL,
    seller_id INT NOT NULL,
    transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('pending', 'completed', 'cancelled', 'disputed') DEFAULT 'pending',
    meeting_location VARCHAR(255),
    meeting_date DATE,
    meeting_time TIME,
    final_price DECIMAL(12,2),
    payment_method VARCHAR(50),
    buyer_rating INT CHECK (buyer_rating >= 1 AND buyer_rating <= 5),
    seller_rating INT CHECK (seller_rating >= 1 AND seller_rating <= 5),
    buyer_review TEXT,
    seller_review TEXT,
    notes TEXT,

    FOREIGN KEY (item_id) REFERENCES marketplace_items(id) ON DELETE CASCADE,
    FOREIGN KEY (buyer_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (seller_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_item_id (item_id),
    INDEX idx_buyer_id (buyer_id),
    INDEX idx_seller_id (seller_id),
    INDEX idx_status (status),
    INDEX idx_transaction_date (transaction_date)
);
```

### 12. feed_posts
```sql
CREATE TABLE feed_posts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    club_id INT NULL, -- For club announcements
    type ENUM('match_report', 'event', 'achievement', 'club_match_announce',
              'club_recruitment', 'tournament_announce', 'training_tip') NOT NULL,
    content TEXT NOT NULL,
    image VARCHAR(500) NULL,
    location VARCHAR(255) NULL,
    event_date DATE NULL,
    event_time TIME NULL,
    max_players INT NULL,
    current_players INT DEFAULT 0,
    tournament_id INT NULL,
    match_id INT NULL,
    badge VARCHAR(100) NULL,
    category VARCHAR(100) NULL,
    score VARCHAR(20) NULL,
    opponent VARCHAR(100) NULL,
    venue VARCHAR(255) NULL,
    old_rating INT NULL,
    new_rating INT NULL,
    likes_count INT DEFAULT 0,
    comments_count INT DEFAULT 0,
    shares_count INT DEFAULT 0,
    is_pinned BOOLEAN DEFAULT FALSE,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (club_id) REFERENCES clubs(id) ON DELETE CASCADE,
    FOREIGN KEY (tournament_id) REFERENCES tournaments(id) ON DELETE SET NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_club_id (club_id),
    INDEX idx_type (type),
    INDEX idx_created_at (created_at),
    INDEX idx_is_pinned (is_pinned),
    INDEX idx_is_verified (is_verified)
);
```

### 13. feed_likes
```sql
CREATE TABLE feed_likes (
    id INT PRIMARY KEY AUTO_INCREMENT,
    post_id INT NOT NULL,
    user_id INT NOT NULL,
    reaction_type ENUM('like', 'love', 'wow', 'support') DEFAULT 'like',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (post_id) REFERENCES feed_posts(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_post_user (post_id, user_id),
    INDEX idx_post_id (post_id),
    INDEX idx_user_id (user_id),
    INDEX idx_reaction_type (reaction_type)
);
```

### 14. feed_comments
```sql
CREATE TABLE feed_comments (
    id INT PRIMARY KEY AUTO_INCREMENT,
    post_id INT NOT NULL,
    user_id INT NOT NULL,
    parent_id INT NULL, -- For nested comments
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (post_id) REFERENCES feed_posts(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_id) REFERENCES feed_comments(id) ON DELETE CASCADE,
    INDEX idx_post_id (post_id),
    INDEX idx_user_id (user_id),
    INDEX idx_parent_id (parent_id)
);
```

### 15. notifications
```sql
CREATE TABLE notifications (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    type ENUM('match_request', 'tournament', 'achievement', 'friend_request',
              'club_invitation', 'marketplace', 'system') NOT NULL,
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    data JSON, -- Additional data as JSON
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id),
    INDEX idx_type (type),
    INDEX idx_is_read (is_read),
    INDEX idx_created_at (created_at)
);
```

### 16. user_reviews
```sql
CREATE TABLE user_reviews (
    id INT PRIMARY KEY AUTO_INCREMENT,
    reviewer_id INT NOT NULL,
    reviewed_id INT NOT NULL,
    rating INT NOT NULL CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    context ENUM('match', 'tournament', 'club', 'marketplace') NOT NULL,
    context_id INT NULL, -- ID of related match/tournament/etc
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (reviewer_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (reviewed_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_reviewer_id (reviewer_id),
    INDEX idx_reviewed_id (reviewed_id),
    INDEX idx_rating (rating),
    INDEX idx_context (context)
);
```

## Indexes and Performance

### Additional Indexes for Performance:
```sql
-- Complex query optimizations
CREATE INDEX idx_users_location_skill ON users(location, skill_level);
CREATE INDEX idx_tournaments_date_status ON tournaments(tournament_date, status);
CREATE INDEX idx_marketplace_category_price ON marketplace_items(category, price);
CREATE INDEX idx_feed_posts_user_date ON feed_posts(user_id, created_at);
CREATE INDEX idx_venue_reviews_venue_rating ON venue_reviews(venue_id, rating);

-- Full-text search indexes
CREATE FULLTEXT INDEX idx_users_name ON users(name);
CREATE FULLTEXT INDEX idx_clubs_name ON clubs(name);
CREATE FULLTEXT INDEX idx_venues_name ON venues(name);
CREATE FULLTEXT INDEX idx_marketplace_name ON marketplace_items(name, description);
CREATE FULLTEXT INDEX idx_feed_content ON feed_posts(content);
```

## Data Relationships

### User Relationships:
- Users can join multiple clubs (many-to-many via club_members)
- Users can participate in multiple tournaments
- Users can create multiple marketplace listings
- Users can post multiple feed posts
- Users can review venues and other users

### Club Relationships:
- Clubs can have multiple members
- Clubs can organize multiple tournaments
- Clubs can post announcements
- Clubs have associated venues

### Tournament Relationships:
- Tournaments have multiple participants
- Tournaments generate multiple matches
- Tournaments can be organized by clubs or individuals
- Tournaments have associated venues

## Migration Strategy

### Initial Migration (v1.0):
1. Create all core tables
2. Add initial indexes
3. Insert seed data for development

### Future Migrations:
- Add new tables as features expand
- Modify existing tables for new requirements
- Add performance indexes based on query analysis
- Implement data partitioning for large tables

## Backup and Recovery

### Backup Strategy:
- Daily automated backups of all tables
- Point-in-time recovery capability
- Encrypted backup storage
- Backup verification procedures

### Data Retention:
- User data: Indefinite (with GDPR compliance)
- Tournament data: 5 years
- Feed posts: 2 years
- Transaction data: 7 years (legal requirement)
- Logs: 1 year