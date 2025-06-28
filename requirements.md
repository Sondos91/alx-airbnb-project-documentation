# Backend Requirements Specifications for Airbnb Clone

## Overview
This document provides detailed technical specifications for three core backend features of the Airbnb clone project: User Authentication, Property Management, and Booking System. Each specification includes API endpoints, data models, validation rules, and performance requirements.

## Table of Contents
1. [User Authentication System](#user-authentication-system)
2. [Property Management System](#property-management-system)
3. [Booking System](#booking-system)

---

## User Authentication System

### Overview
The authentication system provides secure user registration, login, and session management using JWT tokens with refresh token rotation.

### Database Schema

#### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone_number VARCHAR(20),
    profile_photo_url VARCHAR(500),
    is_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    role ENUM('guest', 'host', 'admin') DEFAULT 'guest',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    email_verification_token VARCHAR(255),
    password_reset_token VARCHAR(255),
    password_reset_expires TIMESTAMP
);
```

#### Refresh Tokens Table
```sql
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_revoked BOOLEAN DEFAULT FALSE
);
```

### API Endpoints

#### 1. User Registration
```
POST /api/auth/register
```

**Request Body:**
```json
{
    "email": "user@example.com",
    "password": "SecurePassword123!",
    "confirmPassword": "SecurePassword123!",
    "firstName": "John",
    "lastName": "Doe",
    "phoneNumber": "+1234567890"
}
```

**Validation Rules:**
- Email: Valid email format, unique in database
- Password: Minimum 8 characters, at least 1 uppercase, 1 lowercase, 1 number, 1 special character
- Confirm Password: Must match password
- First Name: 2-50 characters, alphanumeric and spaces only
- Last Name: 2-50 characters, alphanumeric and spaces only
- Phone Number: Valid international format (optional)

**Success Response (201 Created):**
```json
{
    "status": "success",
    "message": "User registered successfully",
    "data": {
        "user": {
            "id": "550e8400-e29b-41d4-a716-446655440000",
            "email": "user@example.com",
            "firstName": "John",
            "lastName": "Doe",
            "isVerified": false,
            "role": "guest"
        },
        "tokens": {
            "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
            "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
            "expiresIn": 3600
        }
    }
}
```

**Error Responses:**
- `400 Bad Request`: Validation errors
- `409 Conflict`: Email already exists
- `429 Too Many Requests`: Rate limit exceeded

#### 2. User Login
```
POST /api/auth/login
```

**Request Body:**
```json
{
    "email": "user@example.com",
    "password": "SecurePassword123!"
}
```

**Success Response (200 OK):**
```json
{
    "status": "success",
    "message": "Login successful",
    "data": {
        "user": {
            "id": "550e8400-e29b-41d4-a716-446655440000",
            "email": "user@example.com",
            "firstName": "John",
            "lastName": "Doe",
            "role": "guest"
        },
        "tokens": {
            "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
            "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
            "expiresIn": 3600
        }
    }
}
```

#### 3. Token Refresh
```
POST /api/auth/refresh
```

**Request Body:**
```json
{
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### 4. User Logout
```
POST /api/auth/logout
```

**Headers:**
```
Authorization: Bearer <access_token>
```

### Security Requirements
- Password hashing using Argon2id with salt
- JWT tokens with 1-hour expiration
- Refresh tokens with 7-day expiration
- Rate limiting: 5 requests per minute for login/register
- Input sanitization and SQL injection protection
- CORS configuration for frontend domains

### Performance Criteria
- Registration/Login response time: < 500ms
- Token validation: < 100ms
- Database queries optimized with indexes on email and user_id
- Redis caching for frequently accessed user data

---

## Property Management System

### Overview
The property management system allows hosts to create, update, and manage property listings with comprehensive details, amenities, and availability.

### Database Schema

#### Properties Table
```sql
CREATE TABLE properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    host_id UUID REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    description TEXT NOT NULL,
    property_type ENUM('apartment', 'house', 'villa', 'cabin', 'unique') NOT NULL,
    room_type ENUM('entire_place', 'private_room', 'shared_room') NOT NULL,
    max_guests INTEGER NOT NULL CHECK (max_guests > 0 AND max_guests <= 20),
    bedrooms INTEGER NOT NULL CHECK (bedrooms > 0),
    bathrooms INTEGER NOT NULL CHECK (bathrooms > 0),
    price_per_night DECIMAL(10,2) NOT NULL CHECK (price_per_night > 0),
    cleaning_fee DECIMAL(10,2) DEFAULT 0,
    service_fee_percentage DECIMAL(5,2) DEFAULT 10.00,
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    country VARCHAR(100) NOT NULL,
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    instant_bookable BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Property Amenities Table
```sql
CREATE TABLE property_amenities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID REFERENCES properties(id) ON DELETE CASCADE,
    amenity_name VARCHAR(100) NOT NULL,
    category ENUM('essentials', 'features', 'location', 'safety') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Property Images Table
```sql
CREATE TABLE property_images (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID REFERENCES properties(id) ON DELETE CASCADE,
    image_url VARCHAR(500) NOT NULL,
    image_type ENUM('main', 'gallery', 'thumbnail') DEFAULT 'gallery',
    display_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### API Endpoints

#### 1. Create Property
```
POST /api/properties
```

**Headers:**
```
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

**Request Body (Form Data):**
```json
{
    "title": "Cozy Downtown Apartment",
    "description": "Beautiful 2-bedroom apartment in the heart of downtown...",
    "propertyType": "apartment",
    "roomType": "entire_place",
    "maxGuests": 4,
    "bedrooms": 2,
    "bathrooms": 1,
    "pricePerNight": 150.00,
    "cleaningFee": 50.00,
    "addressLine1": "123 Main Street",
    "city": "New York",
    "state": "NY",
    "postalCode": "10001",
    "country": "USA",
    "instantBookable": true,
    "amenities": ["wifi", "kitchen", "parking", "air_conditioning"],
    "images": [File1, File2, File3]
}
```

**Validation Rules:**
- Title: 10-200 characters
- Description: 50-2000 characters
- Price: $10-$10,000 per night
- Max guests: 1-20
- Bedrooms/Bathrooms: 1-20
- Address: Valid format for each country
- Images: Max 10 images, each < 10MB, JPG/PNG only

**Success Response (201 Created):**
```json
{
    "status": "success",
    "message": "Property created successfully",
    "data": {
        "property": {
            "id": "550e8400-e29b-41d4-a716-446655440000",
            "title": "Cozy Downtown Apartment",
            "description": "Beautiful 2-bedroom apartment...",
            "propertyType": "apartment",
            "roomType": "entire_place",
            "maxGuests": 4,
            "bedrooms": 2,
            "bathrooms": 1,
            "pricePerNight": 150.00,
            "cleaningFee": 50.00,
            "address": {
                "line1": "123 Main Street",
                "city": "New York",
                "state": "NY",
                "postalCode": "10001",
                "country": "USA"
            },
            "images": [
                {
                    "id": "img-1",
                    "url": "https://storage.example.com/property-1-main.jpg",
                    "type": "main"
                }
            ],
            "amenities": ["wifi", "kitchen", "parking"],
            "instantBookable": true,
            "createdAt": "2024-01-01T10:00:00Z"
        }
    }
}
```

#### 2. Get Property Details
```
GET /api/properties/{propertyId}
```

**Success Response (200 OK):**
```json
{
    "status": "success",
    "data": {
        "property": {
            "id": "550e8400-e29b-41d4-a716-446655440000",
            "title": "Cozy Downtown Apartment",
            "description": "Beautiful 2-bedroom apartment...",
            "host": {
                "id": "host-id",
                "firstName": "John",
                "lastName": "Doe",
                "rating": 4.8,
                "responseRate": 95
            },
            "propertyType": "apartment",
            "roomType": "entire_place",
            "maxGuests": 4,
            "bedrooms": 2,
            "bathrooms": 1,
            "pricePerNight": 150.00,
            "cleaningFee": 50.00,
            "serviceFeePercentage": 10.00,
            "address": {
                "line1": "123 Main Street",
                "city": "New York",
                "state": "NY",
                "postalCode": "10001",
                "country": "USA",
                "latitude": 40.7128,
                "longitude": -74.0060
            },
            "images": [...],
            "amenities": [...],
            "reviews": {
                "averageRating": 4.7,
                "totalReviews": 25,
                "recentReviews": [...]
            },
            "availability": {
                "calendar": [...],
                "nextAvailable": "2024-02-01"
            }
        }
    }
}
```

#### 3. Update Property
```
PUT /api/properties/{propertyId}
```

#### 4. Delete Property
```
DELETE /api/properties/{propertyId}
```

#### 5. Get Host Properties
```
GET /api/properties/host/my-properties
```

### File Upload Requirements
- Image storage: AWS S3 or Cloudinary
- Image processing: Automatic resizing and optimization
- Supported formats: JPG, PNG, WebP
- Max file size: 10MB per image
- Image dimensions: Min 800x600, Max 4000x3000

### Performance Criteria
- Property creation: < 2 seconds (including image upload)
- Property retrieval: < 300ms
- Image loading: < 1 second
- Search queries: < 500ms with pagination
- Database indexes on host_id, property_type, city, price_per_night

---

## Booking System

### Overview
The booking system manages reservations, availability calendars, and payment processing for property bookings.

### Database Schema

#### Bookings Table
```sql
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guest_id UUID REFERENCES users(id) ON DELETE CASCADE,
    property_id UUID REFERENCES properties(id) ON DELETE CASCADE,
    host_id UUID REFERENCES users(id) ON DELETE CASCADE,
    check_in_date DATE NOT NULL,
    check_out_date DATE NOT NULL,
    guest_count INTEGER NOT NULL CHECK (guest_count > 0),
    total_nights INTEGER NOT NULL,
    price_per_night DECIMAL(10,2) NOT NULL,
    cleaning_fee DECIMAL(10,2) DEFAULT 0,
    service_fee DECIMAL(10,2) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status ENUM('pending', 'confirmed', 'cancelled', 'completed', 'refunded') DEFAULT 'pending',
    cancellation_policy ENUM('flexible', 'moderate', 'strict') NOT NULL,
    cancellation_reason TEXT,
    special_requests TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    confirmed_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    CHECK (check_out_date > check_in_date),
    CHECK (total_amount = (price_per_night * total_nights) + cleaning_fee + service_fee)
);
```

#### Availability Calendar Table
```sql
CREATE TABLE availability_calendar (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID REFERENCES properties(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    is_available BOOLEAN DEFAULT TRUE,
    price_override DECIMAL(10,2),
    minimum_stay INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(property_id, date)
);
```

### API Endpoints

#### 1. Create Booking
```
POST /api/bookings
```

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request Body:**
```json
{
    "propertyId": "550e8400-e29b-41d4-a716-446655440000",
    "checkInDate": "2024-02-15",
    "checkOutDate": "2024-02-18",
    "guestCount": 2,
    "specialRequests": "Early check-in if possible",
    "paymentMethod": "stripe",
    "paymentToken": "tok_visa"
}
```

**Validation Rules:**
- Check-in date: Must be in the future (minimum 1 day ahead)
- Check-out date: Must be after check-in date
- Guest count: Must not exceed property max_guests
- Date availability: Property must be available for all dates
- Minimum stay: Must meet property minimum stay requirement

**Success Response (201 Created):**
```json
{
    "status": "success",
    "message": "Booking created successfully",
    "data": {
        "booking": {
            "id": "booking-uuid",
            "propertyId": "550e8400-e29b-41d4-a716-446655440000",
            "propertyTitle": "Cozy Downtown Apartment",
            "hostId": "host-uuid",
            "hostName": "John Doe",
            "checkInDate": "2024-02-15",
            "checkOutDate": "2024-02-18",
            "guestCount": 2,
            "totalNights": 3,
            "pricePerNight": 150.00,
            "cleaningFee": 50.00,
            "serviceFee": 45.00,
            "totalAmount": 545.00,
            "status": "confirmed",
            "paymentStatus": "paid",
            "specialRequests": "Early check-in if possible",
            "createdAt": "2024-01-01T10:00:00Z",
            "confirmedAt": "2024-01-01T10:01:00Z"
        },
        "payment": {
            "transactionId": "txn_123456789",
            "amount": 545.00,
            "currency": "USD",
            "status": "succeeded"
        }
    }
}
```

#### 2. Get Booking Details
```
GET /api/bookings/{bookingId}
```

#### 3. Cancel Booking
```
POST /api/bookings/{bookingId}/cancel
```

**Request Body:**
```json
{
    "reason": "Change of plans",
    "refundAmount": 272.50
}
```

#### 4. Get Property Availability
```
GET /api/properties/{propertyId}/availability
```

**Query Parameters:**
- `startDate`: YYYY-MM-DD
- `endDate`: YYYY-MM-DD

**Success Response (200 OK):**
```json
{
    "status": "success",
    "data": {
        "propertyId": "550e8400-e29b-41d4-a716-446655440000",
        "availability": [
            {
                "date": "2024-02-15",
                "isAvailable": true,
                "price": 150.00
            },
            {
                "date": "2024-02-16",
                "isAvailable": false,
                "price": null
            }
        ],
        "minimumStay": 2,
        "instantBookable": true
    }
}
```

#### 5. Get User Bookings
```
GET /api/bookings/user/{userId}
```

**Query Parameters:**
- `status`: pending, confirmed, cancelled, completed
- `page`: 1, 2, 3...
- `limit`: 10, 20, 50

### Business Logic

#### Booking Calculation
```javascript
const totalNights = differenceInDays(checkOutDate, checkInDate);
const subtotal = pricePerNight * totalNights;
const serviceFee = (subtotal + cleaningFee) * (serviceFeePercentage / 100);
const totalAmount = subtotal + cleaningFee + serviceFee;
```

#### Cancellation Policies
- **Flexible**: Full refund if cancelled 24+ hours before check-in
- **Moderate**: Full refund if cancelled 5+ days before check-in
- **Strict**: 50% refund if cancelled 7+ days before check-in

### Payment Integration
- **Stripe**: Primary payment processor
- **PayPal**: Alternative payment method
- **Multi-currency**: USD, EUR, GBP, CAD, AUD
- **Automatic payouts**: Host receives payment 24 hours after check-in

### Performance Criteria
- Booking creation: < 3 seconds (including payment processing)
- Availability check: < 200ms
- Booking retrieval: < 300ms
- Calendar queries: < 500ms for 30-day range
- Payment processing: < 5 seconds
- Database indexes on guest_id, property_id, check_in_date, status

### Error Handling
- **400 Bad Request**: Invalid dates, guest count, or payment info
- **409 Conflict**: Property not available for selected dates
- **402 Payment Required**: Payment processing failed
- **403 Forbidden**: User not authorized to access booking
- **404 Not Found**: Booking or property not found

---

## General System Requirements

### API Standards
- **RESTful design** with consistent naming conventions
- **JSON request/response** format
- **HTTP status codes** following REST standards
- **Pagination** for list endpoints (limit/offset or cursor-based)
- **Rate limiting** per endpoint and user
- **API versioning** (v1, v2, etc.)

### Security Requirements
- **HTTPS only** for all API communications
- **JWT authentication** with proper token validation
- **Input validation** and sanitization
- **SQL injection protection** using parameterized queries
- **XSS protection** with proper content-type headers
- **CORS configuration** for frontend domains

### Performance Requirements
- **Response times**: < 500ms for 95% of requests
- **Database queries**: Optimized with proper indexing
- **Caching**: Redis for frequently accessed data
- **Load balancing**: Horizontal scaling capability
- **Monitoring**: Real-time performance metrics

### Testing Requirements
- **Unit tests**: 90% code coverage minimum
- **Integration tests**: All API endpoints
- **Load testing**: 1000+ concurrent users
- **Security testing**: OWASP Top 10 compliance
- **API documentation**: OpenAPI/Swagger specification

