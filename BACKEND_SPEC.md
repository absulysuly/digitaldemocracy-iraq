# 🔧 BACKEND API SPECIFICATION
## Digital Democracy Iraq - Complete Backend Requirements

**Version:** 1.0  
**Date:** 2025-11-07  
**Target:** Railway Deployment  
**Database:** PostgreSQL + Prisma ORM

---

## 📋 OVERVIEW

This document specifies all backend API endpoints required for the Digital Democracy Iraq platform. The backend serves two main purposes:

1. **Election Data API:** Candidate information, statistics, governorates
2. **Social Platform API:** Posts, comments, likes, user authentication

---

## 🗄️ DATABASE SCHEMA

### Prisma Schema (`prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ============================================
// ELECTION DATA MODELS
// ============================================

model Candidate {
  id            String   @id @default(uuid())
  name          String
  name_ar       String?
  name_ku       String?
  age           Int?
  gender        Gender?
  education     String?
  occupation    String?
  party         String
  party_ar      String?
  party_ku      String?
  province      String
  province_ar   String?
  province_ku   String?
  constituency  String?
  ballot_number Int?
  photo         String?
  biography     String?
  biography_ar  String?
  biography_ku  String?
  manifesto_url String?
  verified      Boolean  @default(false)
  created_at    DateTime @default(now())
  updated_at    DateTime @updatedAt
  
  @@index([province])
  @@index([party])
  @@index([gender])
}

model Governorate {
  id          String   @id @default(uuid())
  name        String   @unique
  name_ar     String
  name_ku     String
  population  Int?
  region      Region
  capital     String?
  created_at  DateTime @default(now())
}

model Party {
  id         String   @id @default(uuid())
  name       String   @unique
  name_ar    String
  name_ku    String
  logo       String?
  ideology   String?
  founded    Int?
  created_at DateTime @default(now())
}

// ============================================
// USER & SOCIAL MODELS
// ============================================

model User {
  id            String    @id @default(uuid())
  username      String    @unique
  email         String    @unique
  password_hash String
  full_name     String?
  avatar        String?
  bio           String?
  verified      Boolean   @default(false)
  role          UserRole  @default(USER)
  governorate   String?
  created_at    DateTime  @default(now())
  updated_at    DateTime  @updatedAt
  
  posts         Post[]
  comments      Comment[]
  likes         Like[]
  followers     Follow[]  @relation("following")
  following     Follow[]  @relation("follower")
  
  @@index([username])
  @@index([email])
}

model Post {
  id         String   @id @default(uuid())
  content    String
  image_url  String?
  author_id  String
  author     User     @relation(fields: [author_id], references: [id], onDelete: Cascade)
  created_at DateTime @default(now())
  updated_at DateTime @updatedAt
  
  likes      Like[]
  comments   Comment[]
  
  @@index([author_id])
  @@index([created_at])
}

model Comment {
  id         String   @id @default(uuid())
  content    String
  author_id  String
  post_id    String
  author     User     @relation(fields: [author_id], references: [id], onDelete: Cascade)
  post       Post     @relation(fields: [post_id], references: [id], onDelete: Cascade)
  created_at DateTime @default(now())
  
  @@index([post_id])
  @@index([author_id])
}

model Like {
  id         String   @id @default(uuid())
  user_id    String
  post_id    String
  user       User     @relation(fields: [user_id], references: [id], onDelete: Cascade)
  post       Post     @relation(fields: [post_id], references: [id], onDelete: Cascade)
  created_at DateTime @default(now())
  
  @@unique([user_id, post_id])
  @@index([post_id])
}

model Follow {
  id           String   @id @default(uuid())
  follower_id  String
  following_id String
  follower     User     @relation("follower", fields: [follower_id], references: [id], onDelete: Cascade)
  following    User     @relation("following", fields: [following_id], references: [id], onDelete: Cascade)
  created_at   DateTime @default(now())
  
  @@unique([follower_id, following_id])
  @@index([follower_id])
  @@index([following_id])
}

// ============================================
// ENUMS
// ============================================

enum Gender {
  male
  female
}

enum Region {
  north
  south
  central
  kurdistan
}

enum UserRole {
  USER
  MODERATOR
  ADMIN
}
```

---

## 🔌 API ENDPOINTS

### Base URL

```
Production:  https://hamlet-unified-complete-2027-production.up.railway.app
Development: http://localhost:10000
```

### Response Format

All endpoints return JSON in this format:

**Success Response:**
```json
{
  "success": true,
  "data": { ... },
  "message": "Optional success message"
}
```

**Error Response:**
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message"
  }
}
```

---

## 📚 ENDPOINT SPECIFICATIONS

### 1. HEALTH & STATUS

#### `GET /health`

Health check endpoint for monitoring.

**Response:**
```json
{
  "status": "ok",
  "timestamp": "2025-11-07T10:30:00Z",
  "database": "connected",
  "version": "1.0.0"
}
```

---

### 2. AUTHENTICATION

#### `POST /api/auth/register`

Register a new user account.

**Request Body:**
```json
{
  "username": "ahmad_ali",
  "email": "ahmad@example.com",
  "password": "SecurePass123!",
  "full_name": "Ahmad Ali"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid",
      "username": "ahmad_ali",
      "email": "ahmad@example.com",
      "full_name": "Ahmad Ali",
      "avatar": null,
      "verified": false,
      "created_at": "2025-11-07T10:30:00Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

**Validation Rules:**
- Username: 3-20 characters, alphanumeric + underscore
- Email: Valid email format
- Password: Min 8 characters, at least 1 uppercase, 1 lowercase, 1 number

---

#### `POST /api/auth/login`

Authenticate existing user.

**Request Body:**
```json
{
  "username": "ahmad_ali",
  "password": "SecurePass123!"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "user": { ... },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

---

#### `GET /api/auth/me`

Get current authenticated user info.

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "username": "ahmad_ali",
    "email": "ahmad@example.com",
    "full_name": "Ahmad Ali",
    "avatar": "https://...",
    "bio": "Passionate about Iraqi democracy",
    "verified": false,
    "governorate": "Baghdad",
    "created_at": "2025-11-07T10:30:00Z"
  }
}
```

---

#### `POST /api/auth/logout`

Invalidate user token (optional - JWT is stateless).

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

---

### 3. CANDIDATES

#### `GET /api/candidates`

Get paginated list of candidates with optional filters.

**Query Parameters:**
```
page        (number)  Page number (default: 1)
limit       (number)  Items per page (default: 12, max: 100)
search      (string)  Search by name (supports Arabic/Kurdish)
province    (string)  Filter by province
party       (string)  Filter by party
gender      (string)  Filter by gender (male/female)
constituency (string) Filter by constituency
```

**Example Request:**
```
GET /api/candidates?page=1&limit=12&province=Baghdad&gender=female
```

**Response:**
```json
{
  "success": true,
  "data": {
    "candidates": [
      {
        "id": "uuid",
        "name": "Sara Ahmed",
        "name_ar": "سارة أحمد",
        "name_ku": "سارا احمەد",
        "age": 42,
        "gender": "female",
        "education": "PhD in Political Science",
        "occupation": "University Professor",
        "party": "Progressive Alliance",
        "party_ar": "التحالف التقدمي",
        "province": "Baghdad",
        "province_ar": "بغداد",
        "constituency": "District 3",
        "ballot_number": 127,
        "photo": "https://...",
        "biography": "Dr. Sara Ahmed has...",
        "verified": true
      }
    ],
    "pagination": {
      "total": 245,
      "page": 1,
      "limit": 12,
      "pages": 21
    }
  }
}
```

---

#### `GET /api/candidates/:id`

Get single candidate by ID.

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "Sara Ahmed",
    // ... all candidate fields ...
    "created_at": "2025-10-01T00:00:00Z",
    "updated_at": "2025-10-15T12:30:00Z"
  }
}
```

**Error (404):**
```json
{
  "success": false,
  "error": {
    "code": "CANDIDATE_NOT_FOUND",
    "message": "Candidate with ID 'xyz' not found"
  }
}
```

---

### 4. GOVERNORATES

#### `GET /api/governorates`

Get list of all Iraqi governorates.

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "name": "Baghdad",
      "name_ar": "بغداد",
      "name_ku": "بەغدا",
      "population": 8126755,
      "region": "central",
      "capital": "Baghdad"
    },
    {
      "id": "uuid",
      "name": "Basra",
      "name_ar": "البصرة",
      "name_ku": "بەسرە",
      "population": 2600000,
      "region": "south",
      "capital": "Basra"
    }
    // ... more governorates
  ]
}
```

---

#### `GET /api/governorates/:id/candidates`

Get all candidates from a specific governorate.

**Query Parameters:**
```
page   (number)  Page number
limit  (number)  Items per page
```

**Response:**
```json
{
  "success": true,
  "data": {
    "governorate": {
      "id": "uuid",
      "name": "Baghdad",
      "name_ar": "بغداد"
    },
    "candidates": [ ... ],
    "pagination": { ... }
  }
}
```

---

### 5. PARTIES

#### `GET /api/parties`

Get list of all political parties.

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "name": "Progressive Alliance",
      "name_ar": "التحالف التقدمي",
      "name_ku": "یەکێتی پێشکەوتن",
      "logo": "https://...",
      "ideology": "Social Democracy",
      "founded": 2005
    }
    // ... more parties
  ]
}
```

---

### 6. STATISTICS

#### `GET /api/stats`

Get election statistics overview.

**Response:**
```json
{
  "success": true,
  "data": {
    "total_candidates": 7769,
    "total_parties": 142,
    "total_governorates": 18,
    "last_updated": "2025-11-07T10:30:00Z",
    "gender_distribution": {
      "Male": 5512,
      "Female": 2257
    },
    "candidates_per_governorate": [
      {
        "governorate_name": "Baghdad",
        "governorate_name_ar": "بغداد",
        "candidate_count": 1247
      },
      {
        "governorate_name": "Basra",
        "governorate_name_ar": "البصرة",
        "candidate_count": 523
      }
      // ... more
    ],
    "candidates_per_party": [
      {
        "party_name": "Progressive Alliance",
        "candidate_count": 234
      }
      // ... more (top 10)
    ]
  }
}
```

---

### 7. SOCIAL - POSTS

#### `GET /api/posts`

Get paginated feed of posts.

**Query Parameters:**
```
page   (number)  Page number (default: 1)
limit  (number)  Items per page (default: 10)
author (string)  Filter by author username
```

**Headers:**
```
Authorization: Bearer <token>  (optional - for personalized feed)
```

**Response:**
```json
{
  "success": true,
  "data": {
    "posts": [
      {
        "id": "uuid",
        "content": "Excited about the upcoming elections! Every vote counts.",
        "image_url": "https://...",
        "author": {
          "id": "uuid",
          "username": "ahmad_ali",
          "full_name": "Ahmad Ali",
          "avatar": "https://...",
          "verified": false
        },
        "like_count": 42,
        "comment_count": 8,
        "liked_by_user": true,
        "created_at": "2025-11-07T09:30:00Z",
        "updated_at": "2025-11-07T09:30:00Z"
      }
    ],
    "pagination": {
      "total": 1543,
      "page": 1,
      "limit": 10,
      "pages": 155
    }
  }
}
```

---

#### `POST /api/posts`

Create a new post.

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "content": "This is my post content. Maximum 500 characters.",
  "image_url": "https://..." // optional
}
```

**Validation:**
- Content: 1-500 characters
- Image URL: Valid URL format (optional)

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "content": "This is my post content...",
    "image_url": null,
    "author": { ... },
    "like_count": 0,
    "comment_count": 0,
    "created_at": "2025-11-07T10:35:00Z"
  }
}
```

---

#### `GET /api/posts/:id`

Get single post by ID.

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "content": "...",
    "image_url": "...",
    "author": { ... },
    "like_count": 42,
    "comment_count": 8,
    "liked_by_user": true,
    "created_at": "2025-11-07T09:30:00Z"
  }
}
```

---

#### `DELETE /api/posts/:id`

Delete own post (author only).

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "message": "Post deleted successfully"
}
```

**Error (403):**
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "You can only delete your own posts"
  }
}
```

---

### 8. SOCIAL - LIKES

#### `POST /api/posts/:id/like`

Toggle like on a post (like if not liked, unlike if already liked).

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "liked": true,
    "like_count": 43
  }
}
```

---

#### `GET /api/posts/:id/likes`

Get list of users who liked a post.

**Query Parameters:**
```
page   (number)  Page number
limit  (number)  Items per page (default: 20)
```

**Response:**
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": "uuid",
        "username": "ahmad_ali",
        "full_name": "Ahmad Ali",
        "avatar": "https://...",
        "verified": false
      }
    ],
    "pagination": { ... }
  }
}
```

---

### 9. SOCIAL - COMMENTS

#### `GET /api/posts/:id/comments`

Get comments for a post.

**Query Parameters:**
```
page   (number)  Page number
limit  (number)  Items per page (default: 20)
```

**Response:**
```json
{
  "success": true,
  "data": {
    "comments": [
      {
        "id": "uuid",
        "content": "Great point! I totally agree.",
        "author": {
          "id": "uuid",
          "username": "sara_m",
          "full_name": "Sara Mohammed",
          "avatar": "https://...",
          "verified": true
        },
        "created_at": "2025-11-07T09:35:00Z"
      }
    ],
    "pagination": { ... }
  }
}
```

---

#### `POST /api/posts/:id/comments`

Add a comment to a post.

**Headers:**
```
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "content": "This is my comment. Maximum 300 characters."
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "content": "This is my comment...",
    "author": { ... },
    "created_at": "2025-11-07T10:40:00Z"
  }
}
```

---

#### `DELETE /api/comments/:id`

Delete own comment.

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "message": "Comment deleted successfully"
}
```

---

### 10. USERS

#### `GET /api/users/:username`

Get public user profile.

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "username": "ahmad_ali",
    "full_name": "Ahmad Ali",
    "avatar": "https://...",
    "bio": "Passionate about Iraqi democracy",
    "verified": false,
    "governorate": "Baghdad",
    "follower_count": 342,
    "following_count": 189,
    "post_count": 47,
    "created_at": "2025-10-01T00:00:00Z"
  }
}
```

---

#### `PATCH /api/users/:id`

Update own user profile.

**Headers:**
```
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "full_name": "Ahmad Ali Hussein",
  "bio": "Updated bio text",
  "avatar": "https://new-avatar-url.com/image.jpg",
  "governorate": "Basra"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    // ... updated user fields
  }
}
```

---

## 🔐 AUTHENTICATION & AUTHORIZATION

### JWT Token Structure

```json
{
  "user_id": "uuid",
  "username": "ahmad_ali",
  "role": "USER",
  "iat": 1699350000,
  "exp": 1699436400
}
```

### Token Expiration

- Access Token: 24 hours
- Refresh Token: 30 days (future feature)

### Protected Routes

Routes requiring authentication:
- `POST /api/posts`
- `DELETE /api/posts/:id`
- `POST /api/posts/:id/like`
- `POST /api/posts/:id/comments`
- `DELETE /api/comments/:id`
- `PATCH /api/users/:id`
- `GET /api/auth/me`

---

## 📊 RATE LIMITING

| Endpoint Category | Rate Limit | Window |
|------------------|------------|---------|
| Authentication | 5 requests | 15 min |
| Read (GET) | 100 requests | 15 min |
| Write (POST/PATCH/DELETE) | 30 requests | 15 min |
| AI Generation | 10 requests | 1 hour |

**Rate Limit Response (429):**
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again in 5 minutes.",
    "retry_after": 300
  }
}
```

---

## 🚨 ERROR CODES

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Invalid request data |
| `UNAUTHORIZED` | 401 | Missing or invalid token |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Resource already exists |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server error |

---

## 🧪 TESTING

### Sample cURL Commands

**Register User:**
```bash
curl -X POST https://api.example.com/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "test_user",
    "email": "test@example.com",
    "password": "SecurePass123!",
    "full_name": "Test User"
  }'
```

**Get Candidates:**
```bash
curl https://api.example.com/api/candidates?page=1&limit=10
```

**Create Post:**
```bash
curl -X POST https://api.example.com/api/posts \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "This is my test post"
  }'
```

---

## 📚 IMPLEMENTATION CHECKLIST

### Phase 2 (Weeks 3-4):
- [ ] Set up Express.js server
- [ ] Configure Prisma + PostgreSQL
- [ ] Implement authentication endpoints
- [ ] Implement candidate endpoints
- [ ] Add CORS middleware
- [ ] Add rate limiting
- [ ] Deploy to Railway

### Phase 3 (Weeks 5-6):
- [ ] Implement post endpoints
- [ ] Implement like endpoints
- [ ] Implement comment endpoints
- [ ] Implement user profile endpoints
- [ ] Add pagination to all list endpoints
- [ ] Add comprehensive error handling
- [ ] Write API tests

### Phase 4 (Week 7):
- [ ] Add content moderation
- [ ] Implement image upload
- [ ] Add search functionality
- [ ] Optimize database queries
- [ ] Add caching layer (Redis)
- [ ] Set up monitoring (Sentry)

---

**Backend Specification Complete** ✅

This spec provides everything needed to build the backend API for Digital Democracy Iraq.
