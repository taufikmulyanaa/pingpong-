# PingPong+ Authentication & Authorization System

## Overview
Comprehensive authentication and authorization system supporting multiple user tiers, secure session management, and role-based access control.

## Authentication Methods

### 1. JWT Token-Based Authentication

#### Token Structure
```javascript
// JWT Payload Structure
{
  "iss": "pingpong-plus",           // Issuer
  "sub": "user_123",               // Subject (user ID)
  "iat": 1638360000,               // Issued at
  "exp": 1638446400,               // Expiration time (24 hours)
  "type": "access",                // Token type
  "tier": "premium_player",        // User tier
  "permissions": ["read", "write"] // User permissions
}
```

#### Token Generation
```php
<?php
class JWTHandler {
    private $secretKey;
    private $algorithm = 'HS256';

    public function __construct($secretKey) {
        $this->secretKey = $secretKey;
    }

    public function generateToken($payload) {
        $header = json_encode([
            'typ' => 'JWT',
            'alg' => $this.algorithm
        ]);

        $payload['iat'] = time();
        $payload['exp'] = time() + (24 * 60 * 60); // 24 hours
        $payloadEncoded = str_replace(['+', '/', '='], ['-', '_', ''], base64_encode(json_encode($payload)));

        $headerEncoded = str_replace(['+', '/', '='], ['-', '_', ''], base64_encode($header));

        $signature = hash_hmac('sha256', $headerEncoded . "." . $payloadEncoded, $this->secretKey, true);
        $signatureEncoded = str_replace(['+', '/', '='], ['-', '_', ''], base64_encode($signature));

        return $headerEncoded . "." . $payloadEncoded . "." . $signatureEncoded;
    }

    public function validateToken($token) {
        $parts = explode('.', $token);
        if (count($parts) !== 3) {
            return false;
        }

        $header = $parts[0];
        $payload = $parts[1];
        $signature = $parts[2];

        // Verify signature
        $expectedSignature = hash_hmac('sha256', $header . "." . $payload, $this->secretKey, true);
        $expectedSignatureEncoded = str_replace(['+', '/', '='], ['-', '_', ''], base64_encode($expectedSignature));

        if (!hash_equals($signature, $expectedSignatureEncoded)) {
            return false;
        }

        // Decode payload
        $payloadDecoded = json_decode(base64_decode(str_replace(['-', '_'], ['+', '/'], $payload)), true);

        // Check expiration
        if ($payloadDecoded['exp'] < time()) {
            return false;
        }

        return $payloadDecoded;
    }
}
?>
```

### 2. Session-Based Authentication

#### Session Management
```php
<?php
class SessionManager {
    private $sessionLifetime = 3600; // 1 hour
    private $cookieName = 'pingpong_session';

    public function __construct() {
        if (session_status() === PHP_SESSION_NONE) {
            session_start();
        }
    }

    public function createSession($userId, $userData = []) {
        $sessionId = bin2hex(random_bytes(32));
        $expiresAt = time() + $this->sessionLifetime;

        $_SESSION['user_id'] = $userId;
        $_SESSION['session_id'] = $sessionId;
        $_SESSION['created_at'] = time();
        $_SESSION['expires_at'] = $expiresAt;
        $_SESSION['user_agent'] = $_SERVER['HTTP_USER_AGENT'];
        $_SESSION['ip_address'] = $this->getClientIP();

        // Store additional user data
        foreach ($userData as $key => $value) {
            $_SESSION[$key] = $value;
        }

        // Set secure cookie
        setcookie($this->cookieName, $sessionId, $expiresAt, '/', '', true, true);

        return $sessionId;
    }

    public function validateSession() {
        if (!isset($_SESSION['user_id']) || !isset($_SESSION['expires_at'])) {
            return false;
        }

        // Check expiration
        if ($_SESSION['expires_at'] < time()) {
            $this->destroySession();
            return false;
        }

        // Check user agent (basic security)
        if ($_SESSION['user_agent'] !== $_SERVER['HTTP_USER_AGENT']) {
            $this->destroySession();
            return false;
        }

        // Extend session if close to expiration
        if ($_SESSION['expires_at'] - time() < 300) { // 5 minutes
            $_SESSION['expires_at'] = time() + $this->sessionLifetime;
        }

        return true;
    }

    public function getCurrentUserId() {
        return $_SESSION['user_id'] ?? null;
    }

    public function getSessionData($key = null) {
        if ($key === null) {
            return $_SESSION;
        }
        return $_SESSION[$key] ?? null;
    }

    public function updateSessionData($key, $value) {
        $_SESSION[$key] = $value;
    }

    public function destroySession() {
        // Clear session data
        session_unset();
        session_destroy();

        // Clear cookie
        setcookie($this->cookieName, '', time() - 3600, '/', '', true, true);
    }

    private function getClientIP() {
        $ipHeaders = [
            'HTTP_CF_CONNECTING_IP',
            'HTTP_CLIENT_IP',
            'HTTP_X_FORWARDED_FOR',
            'HTTP_X_FORWARDED',
            'HTTP_X_CLUSTER_CLIENT_IP',
            'HTTP_FORWARDED_FOR',
            'HTTP_FORWARDED',
            'REMOTE_ADDR'
        ];

        foreach ($ipHeaders as $header) {
            if (!empty($_SERVER[$header])) {
                $ip = $_SERVER[$header];
                if (strpos($ip, ',') !== false) {
                    $ip = trim(explode(',', $ip)[0]);
                }
                if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
                    return $ip;
                }
            }
        }

        return $_SERVER['REMOTE_ADDR'] ?? '127.0.0.1';
    }
}
?>
```

## Authorization System

### 1. Role-Based Access Control (RBAC)

#### User Roles and Permissions
```php
<?php
class AuthorizationManager {
    private $roles = [
        'basic' => [
            'name' => 'Basic User',
            'permissions' => [
                'read_own_profile',
                'update_own_profile',
                'read_public_content',
                'search_users',
                'join_public_clubs',
                'view_venues',
                'buy_marketplace_items',
                'create_feed_posts',
                'comment_on_posts'
            ]
        ],
        'premium_player' => [
            'name' => 'Premium Player',
            'permissions' => [
                // All basic permissions plus:
                'create_tournaments',
                'manage_own_tournaments',
                'sell_marketplace_items',
                'advanced_search',
                'priority_notifications',
                'detailed_analytics',
                'export_data'
            ]
        ],
        'club_admin' => [
            'name' => 'Club Administrator',
            'permissions' => [
                // All premium permissions plus:
                'manage_club',
                'create_club_tournaments',
                'manage_club_members',
                'club_analytics',
                'bulk_operations',
                'advanced_reporting'
            ]
        ],
        'system_admin' => [
            'name' => 'System Administrator',
            'permissions' => [
                // All permissions
                'manage_users',
                'manage_clubs',
                'manage_tournaments',
                'system_analytics',
                'system_configuration',
                'user_support'
            ]
        ]
    ];

    public function hasPermission($userRole, $permission) {
        if (!isset($this->roles[$userRole])) {
            return false;
        }

        return in_array($permission, $this->roles[$userRole]['permissions']);
    }

    public function getRolePermissions($role) {
        return $this->roles[$role]['permissions'] ?? [];
    }

    public function getAllRoles() {
        return array_keys($this->roles);
    }

    public function canCreateTournament($userRole) {
        return $this->hasPermission($userRole, 'create_tournaments');
    }

    public function canManageClub($userRole) {
        return $this->hasPermission($userRole, 'manage_club');
    }

    public function canSellItems($userRole) {
        return $this->hasPermission($userRole, 'sell_marketplace_items');
    }
}
?>
```

### 2. Resource-Based Permissions

#### Tournament Permissions
```php
<?php
class TournamentPermissions {
    public static function canEditTournament($userId, $tournamentId, $db) {
        // Check if user is the organizer
        $stmt = $db->prepare("SELECT organizer_id FROM tournaments WHERE id = ?");
        $stmt->execute([$tournamentId]);
        $tournament = $stmt->fetch(PDO::FETCH_ASSOC);

        if ($tournament && $tournament['organizer_id'] == $userId) {
            return true;
        }

        // Check if user is club admin of the organizing club
        $stmt = $db->prepare("
            SELECT ct.role FROM club_members ct
            JOIN tournaments t ON t.club_id = ct.club_id
            WHERE ct.user_id = ? AND t.id = ? AND ct.role = 'admin'
        ");
        $stmt->execute([$userId, $tournamentId]);

        return $stmt->fetch(PDO::FETCH_ASSOC) !== false;
    }

    public static function canRegisterForTournament($userId, $tournamentId, $db) {
        // Check if tournament is open for registration
        $stmt = $db->prepare("
            SELECT status, max_participants, current_participants
            FROM tournaments WHERE id = ?
        ");
        $stmt->execute([$tournamentId]);
        $tournament = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$tournament || $tournament['status'] !== 'open') {
            return false;
        }

        // Check if tournament is not full
        if ($tournament['current_participants'] >= $tournament['max_participants']) {
            return false;
        }

        // Check if user is not already registered
        $stmt = $db->prepare("
            SELECT id FROM tournament_participants
            WHERE tournament_id = ? AND user_id = ?
        ");
        $stmt->execute([$tournamentId, $userId]);

        return $stmt->fetch(PDO::FETCH_ASSOC) === false;
    }
}
?>
```

#### Club Permissions
```php
<?php
class ClubPermissions {
    public static function canManageClub($userId, $clubId, $db) {
        $stmt = $db->prepare("
            SELECT role FROM club_members
            WHERE user_id = ? AND club_id = ? AND role IN ('admin', 'moderator')
        ");
        $stmt->execute([$userId, $clubId]);

        return $stmt->fetch(PDO::FETCH_ASSOC) !== false;
    }

    public static function canInviteMembers($userId, $clubId, $db) {
        return self::canManageClub($userId, $clubId, $db);
    }

    public static function canPostAnnouncements($userId, $clubId, $db) {
        $stmt = $db->prepare("
            SELECT role FROM club_members
            WHERE user_id = ? AND club_id = ? AND role IN ('admin', 'moderator', 'member')
        ");
        $stmt->execute([$userId, $clubId]);

        return $stmt->fetch(PDO::FETCH_ASSOC) !== false;
    }
}
?>
```

## Security Implementation

### 1. Password Security
```php
<?php
class PasswordSecurity {
    public static function hashPassword($password) {
        return password_hash($password, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,    // 64 MB
            'time_cost' => 4,          // 4 iterations
            'threads' => 3             // 3 parallel threads
        ]);
    }

    public static function verifyPassword($password, $hash) {
        return password_verify($password, $hash);
    }

    public static function needsRehash($hash) {
        return password_needs_rehash($hash, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost' => 4,
            'threads' => 3
        ]);
    }

    public static function generateResetToken() {
        return bin2hex(random_bytes(32));
    }

    public static function validatePasswordStrength($password) {
        $errors = [];

        if (strlen($password) < 8) {
            $errors[] = 'Password must be at least 8 characters long';
        }

        if (!preg_match('/[A-Z]/', $password)) {
            $errors[] = 'Password must contain at least one uppercase letter';
        }

        if (!preg_match('/[a-z]/', $password)) {
            $errors[] = 'Password must contain at least one lowercase letter';
        }

        if (!preg_match('/[0-9]/', $password)) {
            $errors[] = 'Password must contain at least one number';
        }

        if (!preg_match('/[^A-Za-z0-9]/', $password)) {
            $errors[] = 'Password must contain at least one special character';
        }

        return $errors;
    }
}
?>
```

### 2. Rate Limiting
```php
<?php
class RateLimiter {
    private $redis;
    private $maxAttempts;
    private $windowSeconds;

    public function __construct($redis, $maxAttempts = 5, $windowSeconds = 300) {
        $this->redis = $redis;
        $this->maxAttempts = $maxAttempts;
        $this->windowSeconds = $windowSeconds;
    }

    public function checkLimit($key, $action = 'general') {
        $fullKey = "rate_limit:{$action}:{$key}";
        $current = $this->redis->get($fullKey);

        if ($current === false) {
            // First attempt
            $this->redis->setex($fullKey, $this->windowSeconds, 1);
            return true;
        }

        if ($current >= $this->maxAttempts) {
            return false;
        }

        // Increment attempts
        $this->redis->incr($fullKey);
        return true;
    }

    public function getRemainingAttempts($key, $action = 'general') {
        $fullKey = "rate_limit:{$action}:{$key}";
        $current = $this->redis->get($fullKey);

        if ($current === false) {
            return $this->maxAttempts;
        }

        return max(0, $this->maxAttempts - $current);
    }

    public function getResetTime($key, $action = 'general') {
        $fullKey = "rate_limit:{$action}:{$key}";
        return $this->redis->ttl($fullKey);
    }
}

// Usage for login attempts
$rateLimiter = new RateLimiter($redis, 5, 900); // 5 attempts per 15 minutes

if (!$rateLimiter->checkLimit($email, 'login')) {
    $remaining = $rateLimiter->getRemainingAttempts($email, 'login');
    $resetTime = $rateLimiter->getResetTime($email, 'login');

    jsonResponse([
        'error' => 'Too many login attempts',
        'remaining_attempts' => $remaining,
        'reset_in_seconds' => $resetTime
    ], 429);
}
?>
```

### 3. Input Validation and Sanitization
```php
<?php
class InputValidator {
    public static function sanitizeString($input) {
        return htmlspecialchars(trim($input), ENT_QUOTES, 'UTF-8');
    }

    public static function validateEmail($email) {
        return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
    }

    public static function validateUsername($username) {
        // Alphanumeric, underscore, dash, 3-20 characters
        return preg_match('/^[a-zA-Z0-9_-]{3,20}$/', $username);
    }

    public static function validatePhone($phone) {
        // Indonesian phone number format
        return preg_match('/^(\+62|62|0)[8-9][0-9]{7,11}$/', $phone);
    }

    public static function validateEloRating($rating) {
        return is_numeric($rating) && $rating >= 0 && $rating <= 3000;
    }

    public static function validateTournamentFee($fee) {
        return is_numeric($fee) && $fee >= 0 && $fee <= 10000000; // Max 10M IDR
    }

    public static function validateURL($url) {
        return filter_var($url, FILTER_VALIDATE_URL) !== false;
    }

    public static function validateImageFile($file) {
        $allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
        $maxSize = 5 * 1024 * 1024; // 5MB

        if (!in_array($file['type'], $allowedTypes)) {
            return 'Invalid file type. Only JPEG, PNG, GIF, and WebP are allowed.';
        }

        if ($file['size'] > $maxSize) {
            return 'File size too large. Maximum size is 5MB.';
        }

        return true;
    }
}

// Usage in API endpoints
function validateUserRegistration($data) {
    $errors = [];

    if (empty($data['name']) || strlen($data['name']) < 2) {
        $errors['name'] = 'Name must be at least 2 characters long';
    }

    if (!InputValidator::validateEmail($data['email'])) {
        $errors['email'] = 'Invalid email format';
    }

    $passwordErrors = PasswordSecurity::validatePasswordStrength($data['password']);
    if (!empty($passwordErrors)) {
        $errors['password'] = $passwordErrors;
    }

    if (!empty($data['phone']) && !InputValidator::validatePhone($data['phone'])) {
        $errors['phone'] = 'Invalid phone number format';
    }

    if (!empty($data['elo']) && !InputValidator::validateEloRating($data['elo'])) {
        $errors['elo'] = 'Invalid ELO rating';
    }

    return $errors;
}
?>
```

## User Tier System

### 1. Tier Definitions
```php
<?php
class UserTierManager {
    const TIERS = [
        'basic' => [
            'name' => 'Basic',
            'price_monthly' => 0,
            'features' => [
                'max_friends' => 100,
                'search_radius' => 5, // km
                'tournaments_per_month' => 2,
                'marketplace_listings' => 0, // cannot sell
                'analytics' => 'basic',
                'support' => 'community'
            ]
        ],
        'premium_player' => [
            'name' => 'Premium Player',
            'price_monthly' => 30000, // IDR
            'features' => [
                'max_friends' => 500,
                'search_radius' => 25,
                'tournaments_per_month' => 10,
                'marketplace_listings' => 50,
                'analytics' => 'detailed',
                'support' => 'priority'
            ]
        ],
        'club_admin' => [
            'name' => 'Club Administrator',
            'price_monthly' => 150000, // IDR
            'features' => [
                'max_friends' => -1, // unlimited
                'search_radius' => -1, // unlimited
                'tournaments_per_month' => -1, // unlimited
                'marketplace_listings' => -1, // unlimited
                'analytics' => 'advanced',
                'support' => 'dedicated'
            ]
        ]
    ];

    public static function getTierFeatures($tier) {
        return self::TIERS[$tier] ?? self::TIERS['basic'];
    }

    public static function canAccessFeature($tier, $feature, $value = null) {
        $tierFeatures = self::getTierFeatures($tier);

        if (!isset($tierFeatures['features'][$feature])) {
            return false;
        }

        $featureLimit = $tierFeatures['features'][$feature];

        if ($featureLimit === -1) { // Unlimited
            return true;
        }

        if ($value === null) {
            return $featureLimit > 0;
        }

        return $value <= $featureLimit;
    }

    public static function upgradeTier($userId, $newTier, $db) {
        $stmt = $db->prepare("UPDATE users SET tier = ?, updated_at = NOW() WHERE id = ?");
        return $stmt->execute([$newTier, $userId]);
    }

    public static function checkTierLimits($userId, $db) {
        $stmt = $db->prepare("SELECT tier FROM users WHERE id = ?");
        $stmt->execute([$userId]);
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$user) {
            return ['error' => 'User not found'];
        }

        $tier = $user['tier'];
        $features = self::getTierFeatures($tier);

        // Check current usage against limits
        $limits = [];

        // Check friends count
        $stmt = $db->prepare("SELECT COUNT(*) as count FROM user_friends WHERE user_id = ?");
        $stmt->execute([$userId]);
        $friendsCount = $stmt->fetch(PDO::FETCH_ASSOC)['count'];

        if (!self::canAccessFeature($tier, 'max_friends', $friendsCount)) {
            $limits['friends'] = [
                'current' => $friendsCount,
                'limit' => $features['features']['max_friends']
            ];
        }

        // Check tournaments this month
        $stmt = $db->prepare("
            SELECT COUNT(*) as count FROM tournaments
            WHERE organizer_id = ? AND MONTH(created_at) = MONTH(CURRENT_DATE())
            AND YEAR(created_at) = YEAR(CURRENT_DATE())
        ");
        $stmt->execute([$userId]);
        $tournamentsCount = $stmt->fetch(PDO::FETCH_ASSOC)['count'];

        if (!self::canAccessFeature($tier, 'tournaments_per_month', $tournamentsCount)) {
            $limits['tournaments'] = [
                'current' => $tournamentsCount,
                'limit' => $features['features']['tournaments_per_month']
            ];
        }

        return $limits;
    }
}
?>
```

### 2. Feature Gates
```php
<?php
class FeatureGate {
    public static function canCreateTournament($userId, $db) {
        $tierLimits = UserTierManager::checkTierLimits($userId, $db);

        if (isset($tierLimits['tournaments'])) {
            return [
                'allowed' => false,
                'reason' => 'Tournament creation limit reached',
                'current' => $tierLimits['tournaments']['current'],
                'limit' => $tierLimits['tournaments']['limit']
            ];
        }

        return ['allowed' => true];
    }

    public static function canSellItem($userId, $db) {
        $stmt = $db->prepare("SELECT tier FROM users WHERE id = ?");
        $stmt->execute([$userId]);
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        $canSell = UserTierManager::canAccessFeature($user['tier'], 'marketplace_listings');

        if (!$canSell) {
            return [
                'allowed' => false,
                'reason' => 'Selling requires Premium subscription',
                'upgrade_required' => 'premium_player'
            ];
        }

        return ['allowed' => true];
    }

    public static function canSearchExtendedRadius($userId, $db) {
        $stmt = $db->prepare("SELECT tier FROM users WHERE id = ?");
        $stmt->execute([$userId]);
        $user = $stmt->fetch(PDO::FETCH_ASSOC);

        $tier = $user['tier'];
        $maxRadius = UserTierManager::getTierFeatures($tier)['features']['search_radius'];

        return [
            'allowed' => true,
            'max_radius' => $maxRadius
        ];
    }
}
?>
```

## Frontend Authentication

### 1. Auth Service
```javascript
class AuthService {
    constructor() {
        this.baseURL = '/api/v1';
        this.tokenKey = 'auth_token';
        this.refreshTokenKey = 'refresh_token';
    }

    async login(credentials) {
        try {
            const response = await fetch(`${this.baseURL}/auth/login`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(credentials)
            });

            const data = await response.json();

            if (data.success) {
                this.setTokens(data.data.token, data.data.refresh_token);
                return { success: true, user: data.data.user };
            } else {
                return { success: false, error: data.error };
            }
        } catch (error) {
            return { success: false, error: 'Network error' };
        }
    }

    async register(userData) {
        try {
            const response = await fetch(`${this.baseURL}/auth/register`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(userData)
            });

            const data = await response.json();

            if (data.success) {
                this.setTokens(data.data.token, data.data.refresh_token);
                return { success: true, user: data.data.user };
            } else {
                return { success: false, error: data.error };
            }
        } catch (error) {
            return { success: false, error: 'Network error' };
        }
    }

    async logout() {
        try {
            const response = await fetch(`${this.baseURL}/auth/logout`, {
                method: 'DELETE',
                headers: {
                    'Authorization': `Bearer ${this.getToken()}`
                }
            });

            this.clearTokens();
            return { success: true };
        } catch (error) {
            // Even if API call fails, clear local tokens
            this.clearTokens();
            return { success: true };
        }
    }

    async refreshToken() {
        try {
            const refreshToken = this.getRefreshToken();
            if (!refreshToken) {
                return { success: false, error: 'No refresh token' };
            }

            const response = await fetch(`${this.baseURL}/auth/refresh`, {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${refreshToken}`
                }
            });

            const data = await response.json();

            if (data.success) {
                this.setTokens(data.data.token, data.data.refresh_token);
                return { success: true };
            } else {
                this.clearTokens();
                return { success: false, error: data.error };
            }
        } catch (error) {
            this.clearTokens();
            return { success: false, error: 'Network error' };
        }
    }

    getToken() {
        return localStorage.getItem(this.tokenKey);
    }

    getRefreshToken() {
        return localStorage.getItem(this.refreshTokenKey);
    }

    setTokens(accessToken, refreshToken) {
        localStorage.setItem(this.tokenKey, accessToken);
        if (refreshToken) {
            localStorage.setItem(this.refreshTokenKey, refreshToken);
        }
    }

    clearTokens() {
        localStorage.removeItem(this.tokenKey);
        localStorage.removeItem(this.refreshTokenKey);
    }

    isAuthenticated() {
        const token = this.getToken();
        if (!token) return false;

        try {
            const payload = JSON.parse(atob(token.split('.')[1]));
            return payload.exp > Date.now() / 1000;
        } catch (error) {
            return false;
        }
    }

    getUserFromToken() {
        const token = this.getToken();
        if (!token) return null;

        try {
            const payload = JSON.parse(atob(token.split('.')[1]));
            return {
                id: payload.sub,
                tier: payload.tier,
                permissions: payload.permissions
            };
        } catch (error) {
            return null;
        }
    }
}
```

### 2. Auth Middleware
```javascript
class AuthMiddleware {
    static requiresAuth() {
        return (request) => {
            const authService = new AuthService();

            if (!authService.isAuthenticated()) {
                // Redirect to login or show auth modal
                window.location.hash = '#/login';
                throw new Error('Authentication required');
            }

            // Add auth header to request
            request.headers = {
                ...request.headers,
                'Authorization': `Bearer ${authService.getToken()}`
            };

            return request;
        };
    }

    static requiresPermission(permission) {
        return (request) => {
            const authService = new AuthService();
            const user = authService.getUserFromToken();

            if (!user || !user.permissions.includes(permission)) {
                throw new Error(`Permission '${permission}' required`);
            }

            return request;
        };
    }

    static requiresTier(minTier) {
        const tierLevels = {
            'basic': 1,
            'premium_player': 2,
            'club_admin': 3,
            'system_admin': 4
        };

        return (request) => {
            const authService = new AuthService();
            const user = authService.getUserFromToken();

            if (!user || tierLevels[user.tier] < tierLevels[minTier]) {
                throw new Error(`Tier '${minTier}' or higher required`);
            }

            return request;
        };
    }
}

// Usage in API calls
const userService = new UserService();
userService.addMiddleware(AuthMiddleware.requiresAuth());
userService.addMiddleware(AuthMiddleware.requiresTier('premium_player'));
```

This comprehensive authentication and authorization system provides secure, scalable access control for the PingPong+ platform with support for multiple user tiers and granular permissions.