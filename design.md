# Design Document: Spot Share Track

## Overview

Spot Share Track is a real-time peer-to-peer parking marketplace built as a Progressive Web App (PWA) using React, Node.js, Express, Socket.io, and MongoDB. The system architecture prioritizes high-integrity manual workflows, real-time state synchronization, and fraud prevention without relying on AI or expensive third-party APIs.

The core "Verified Handshake" workflow ensures trust through a multi-phase process: Hold (soft-lock reservation), Arrival (PIN generation), Handshake (PIN verification), Payment (UPI direct transfer), and Activation (session start). Real-time updates are delivered via Socket.io, while MongoDB TTL indexes handle automatic expiration of stale reservations.

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer (PWA)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Driver App   │  │  Host App    │  │ Service      │     │
│  │ (React)      │  │  (React)     │  │ Worker       │     │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┘     │
└─────────┼──────────────────┼──────────────────────────────┘
          │                  │
          │  HTTP/REST       │  Socket.io (WebSocket)
          │                  │
┌─────────┼──────────────────┼──────────────────────────────┐
│         ▼                  ▼                                │
│  ┌──────────────────────────────────────────────┐         │
│  │         API Gateway (Express.js)              │         │
│  └──────────────┬───────────────────────────────┘         │
│                 │                                           │
│  ┌──────────────┼───────────────────────────────┐         │
│  │              ▼                                 │         │
│  │  ┌─────────────────┐  ┌──────────────────┐  │         │
│  │  │ Auth Service    │  │ Socket Manager   │  │         │
│  │  └─────────────────┘  └──────────────────┘  │         │
│  │                                               │         │
│  │  ┌─────────────────┐  ┌──────────────────┐  │         │
│  │  │ Spot Service    │  │ Session Service  │  │         │
│  │  └─────────────────┘  └──────────────────┘  │         │
│  │                                               │         │
│  │  ┌─────────────────┐  ┌──────────────────┐  │         │
│  │  │ Hold Service    │  │ Payment Service  │  │         │
│  │  └─────────────────┘  └──────────────────┘  │         │
│  │                                               │         │
│  │         Backend Services (Node.js)           │         │
│  └───────────────────────────────────────────────┘         │
│                         │                                   │
│                         ▼                                   │
│  ┌──────────────────────────────────────────────┐         │
│  │         MongoDB Database                      │         │
│  │  - Users Collection                           │         │
│  │  - Spots Collection (2dsphere index)          │         │
│  │  - Holds Collection (TTL index)               │         │
│  │  - Sessions Collection                        │         │
│  │  - Disputes Collection                        │         │
│  │  - BannedDevices Collection                   │         │
│  └──────────────────────────────────────────────┘         │
│                                                             │
│                  Server Layer                               │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack

- **Frontend**: React 18+ (PWA with Service Worker for offline capability)
- **Backend**: Node.js 18+ with Express.js
- **Real-time**: Socket.io for bidirectional event-based communication
- **Database**: MongoDB 6+ with geospatial indexing
- **Authentication**: JWT tokens with phone number + OTP
- **Device Fingerprinting**: FingerprintJS or custom implementation
- **Payment**: UPI Intent Links (no payment gateway integration)

### Design Principles

1. **Manual Verification First**: Trust is built through human verification (PIN entry) rather than automated systems
2. **Real-time by Default**: All state changes broadcast immediately via WebSockets
3. **Fail-Safe Expiration**: TTL indexes automatically clean up stale data
4. **Device-Level Tracking**: Fraud prevention through device fingerprinting
5. **Stateful Recovery**: Session state persists in database for browser reload scenarios

## Components and Interfaces

### 1. Authentication Service

**Responsibilities:**
- User registration and login
- Device ID generation and validation
- JWT token management
- Banned device checking

**Interface:**

```typescript
interface AuthService {
  // Register new user with phone number
  register(phoneNumber: string, deviceId: string, userType: 'driver' | 'host'): Promise<{
    userId: string;
    token: string;
  }>;
  
  // Send OTP for login
  sendOTP(phoneNumber: string): Promise<{ otpSent: boolean }>;
  
  // Verify OTP and login
  verifyOTP(phoneNumber: string, otp: string, deviceId: string): Promise<{
    userId: string;
    token: string;
  }>;
  
  // Check if device is banned
  isDeviceBanned(deviceId: string): Promise<boolean>;
  
  // Validate JWT token
  validateToken(token: string): Promise<{ userId: string; valid: boolean }>;
}
```

### 2. Spot Service

**Responsibilities:**
- Spot creation and management
- Geospatial search
- Spot availability status
- Photo upload handling

**Interface:**

```typescript
interface SpotService {
  // Create new parking spot
  createSpot(hostId: string, spotData: {
    address: string;
    coordinates: { lat: number; lng: number };
    hourlyRate: number;
    photos: string[];
    availabilitySchedule: AvailabilitySchedule;
  }): Promise<{ spotId: string }>;
  
  // Search spots within radius
  searchSpots(coordinates: { lat: number; lng: number }, radiusMeters: number): Promise<Spot[]>;
  
  // Get spot details
  getSpot(spotId: string): Promise<Spot>;
  
  // Update spot availability
  updateSpotStatus(spotId: string, status: 'available' | 'held' | 'active'): Promise<void>;
  
  // Mark spot as fraudulent
  markSpotFraudulent(spotId: string, reportedBy: string): Promise<void>;
}

interface Spot {
  spotId: string;
  hostId: string;
  address: string;
  location: {
    type: 'Point';
    coordinates: [number, number]; // [longitude, latitude]
  };
  hourlyRate: number;
  photos: string[];
  status: 'available' | 'held' | 'active' | 'suspended';
  availabilitySchedule: AvailabilitySchedule;
  createdAt: Date;
}

interface AvailabilitySchedule {
  monday: { start: string; end: string }[];
  tuesday: { start: string; end: string }[];
  wednesday: { start: string; end: string }[];
  thursday: { start: string; end: string }[];
  friday: { start: string; end: string }[];
  saturday: { start: string; end: string }[];
  sunday: { start: string; end: string }[];
}
```

### 3. Hold Service

**Responsibilities:**
- Create temporary reservations with TTL
- Generate unique PINs
- Validate PINs
- Auto-expire holds

**Interface:**

```typescript
interface HoldService {
  // Create hold with 15-minute TTL
  createHold(driverId: string, spotId: string): Promise<{
    holdId: string;
    pin: string;
    expiresAt: Date;
  }>;
  
  // Verify PIN entered by host
  verifyPIN(spotId: string, pin: string): Promise<{
    valid: boolean;
    holdId?: string;
    driverId?: string;
  }>;
  
  // Get active hold for driver
  getActiveHold(driverId: string): Promise<Hold | null>;
  
  // Cancel hold manually
  cancelHold(holdId: string): Promise<void>;
  
  // Check if driver has active hold
  hasActiveHold(driverId: string): Promise<boolean>;
}

interface Hold {
  holdId: string;
  driverId: string;
  spotId: string;
  pin: string;
  status: 'pending' | 'verified' | 'expired';
  createdAt: Date;
  expiresAt: Date;
  verifiedAt?: Date;
}
```

### 4. Session Service

**Responsibilities:**
- Activate parking sessions
- Track session duration
- Handle session completion
- Manage overstay notifications

**Interface:**

```typescript
interface SessionService {
  // Activate session after payment
  activateSession(holdId: string, upiTransactionId: string): Promise<{
    sessionId: string;
    startTime: Date;
  }>;
  
  // End active session
  endSession(sessionId: string): Promise<{
    endTime: Date;
    duration: number;
    totalCost: number;
  }>;
  
  // Get active session for driver
  getActiveSession(driverId: string): Promise<Session | null>;
  
  // Get all active sessions for host
  getHostActiveSessions(hostId: string): Promise<Session[]>;
  
  // Trigger overstay notification
  reportOverstay(sessionId: string): Promise<void>;
  
  // Get session history
  getSessionHistory(userId: string, userType: 'driver' | 'host'): Promise<Session[]>;
}

interface Session {
  sessionId: string;
  holdId: string;
  driverId: string;
  hostId: string;
  spotId: string;
  upiTransactionId: string;
  status: 'active' | 'completed' | 'disputed';
  startTime: Date;
  endTime?: Date;
  duration?: number;
  totalCost?: number;
  overstayReported: boolean;
  createdAt: Date;
}
```

### 5. Payment Service

**Responsibilities:**
- Generate UPI Intent Links
- Store transaction IDs
- Handle payment disputes

**Interface:**

```typescript
interface PaymentService {
  // Generate UPI Intent Link
  generateUPILink(hostUpiId: string, amount: number, note: string): Promise<{
    upiLink: string;
  }>;
  
  // Record transaction ID from driver
  recordTransaction(sessionId: string, upiTransactionId: string): Promise<void>;
  
  // Create payment dispute
  createDispute(sessionId: string, reportedBy: string, reason: string): Promise<{
    disputeId: string;
  }>;
  
  // Resolve dispute
  resolveDispute(disputeId: string, resolution: string): Promise<void>;
}

interface Dispute {
  disputeId: string;
  sessionId: string;
  reportedBy: string;
  reason: string;
  upiTransactionId: string;
  status: 'open' | 'resolved';
  resolution?: string;
  createdAt: Date;
  resolvedAt?: Date;
}
```

### 6. Socket Manager

**Responsibilities:**
- Manage WebSocket connections
- Broadcast state changes
- Handle reconnection
- Room-based messaging

**Interface:**

```typescript
interface SocketManager {
  // Emit event to specific user
  emitToUser(userId: string, event: string, data: any): void;
  
  // Emit event to all users watching a spot
  emitToSpotWatchers(spotId: string, event: string, data: any): void;
  
  // Broadcast to all connected clients
  broadcast(event: string, data: any): void;
  
  // Join user to spot room
  joinSpotRoom(userId: string, spotId: string): void;
  
  // Leave spot room
  leaveSpotRoom(userId: string, spotId: string): void;
  
  // Handle user disconnect
  handleDisconnect(userId: string): void;
}

// Socket Events
enum SocketEvents {
  SPOT_STATUS_CHANGED = 'spot:status:changed',
  HOLD_CREATED = 'hold:created',
  HOLD_EXPIRED = 'hold:expired',
  PIN_VERIFIED = 'pin:verified',
  SESSION_ACTIVATED = 'session:activated',
  SESSION_ENDED = 'session:ended',
  OVERSTAY_ALERT = 'overstay:alert',
  CONNECTION_RESTORED = 'connection:restored',
}
```

## Data Models

### MongoDB Collections

#### Users Collection

```typescript
interface User {
  _id: ObjectId;
  phoneNumber: string;
  deviceId: string;
  userType: 'driver' | 'host';
  name?: string;
  upiId?: string; // For hosts
  rating: number;
  totalSessions: number;
  accountStatus: 'active' | 'suspended';
  createdAt: Date;
  updatedAt: Date;
}

// Indexes
// - phoneNumber: unique
// - deviceId: non-unique (for tracking)
```

#### Spots Collection

```typescript
interface SpotDocument {
  _id: ObjectId;
  hostId: ObjectId;
  address: string;
  location: {
    type: 'Point';
    coordinates: [number, number]; // [longitude, latitude] - GeoJSON format
  };
  hourlyRate: number;
  photos: string[]; // URLs or base64
  status: 'available' | 'held' | 'active' | 'suspended';
  availabilitySchedule: AvailabilitySchedule;
  createdAt: Date;
  updatedAt: Date;
}

// Indexes
// - location: 2dsphere (geospatial index)
// - hostId: non-unique
// - status: non-unique
```

#### Holds Collection

```typescript
interface HoldDocument {
  _id: ObjectId;
  driverId: ObjectId;
  spotId: ObjectId;
  pin: string; // 4-digit string
  status: 'pending' | 'verified' | 'expired';
  createdAt: Date;
  expiresAt: Date; // TTL index on this field
  verifiedAt?: Date;
}

// Indexes
// - expiresAt: TTL index (expireAfterSeconds: 0)
// - driverId: non-unique
// - spotId: non-unique
// - pin: non-unique
// - status: non-unique
```

#### Sessions Collection

```typescript
interface SessionDocument {
  _id: ObjectId;
  holdId: ObjectId;
  driverId: ObjectId;
  hostId: ObjectId;
  spotId: ObjectId;
  upiTransactionId: string;
  status: 'active' | 'completed' | 'disputed';
  startTime: Date;
  endTime?: Date;
  duration?: number; // in minutes
  totalCost?: number;
  overstayReported: boolean;
  overstayNotificationCount: number;
  createdAt: Date;
  updatedAt: Date;
}

// Indexes
// - driverId: non-unique
// - hostId: non-unique
// - spotId: non-unique
// - status: non-unique
// - upiTransactionId: non-unique
```

#### Disputes Collection

```typescript
interface DisputeDocument {
  _id: ObjectId;
  sessionId: ObjectId;
  reportedBy: ObjectId;
  reason: string;
  upiTransactionId: string;
  status: 'open' | 'resolved';
  resolution?: string;
  createdAt: Date;
  resolvedAt?: Date;
}

// Indexes
// - sessionId: non-unique
// - status: non-unique
```

#### BannedDevices Collection

```typescript
interface BannedDeviceDocument {
  _id: ObjectId;
  deviceId: string;
  userId: ObjectId;
  reason: string;
  bannedAt: Date;
}

// Indexes
// - deviceId: unique
```

## API Specifications

### REST Endpoints

#### Authentication

```
POST /api/auth/register
Body: { phoneNumber, deviceId, userType }
Response: { userId, token }

POST /api/auth/send-otp
Body: { phoneNumber }
Response: { otpSent: boolean }

POST /api/auth/verify-otp
Body: { phoneNumber, otp, deviceId }
Response: { userId, token }

GET /api/auth/validate
Headers: { Authorization: Bearer <token> }
Response: { valid: boolean, userId }
```

#### Spots

```
POST /api/spots
Headers: { Authorization: Bearer <token> }
Body: { address, coordinates, hourlyRate, photos, availabilitySchedule }
Response: { spotId }

GET /api/spots/search?lat=<lat>&lng=<lng>&radius=<meters>
Response: { spots: Spot[] }

GET /api/spots/:spotId
Response: { spot: Spot }

POST /api/spots/:spotId/report-fraud
Headers: { Authorization: Bearer <token> }
Body: { reason }
Response: { reported: boolean }
```

#### Holds

```
POST /api/holds
Headers: { Authorization: Bearer <token> }
Body: { spotId }
Response: { holdId, pin, expiresAt }

POST /api/holds/verify-pin
Headers: { Authorization: Bearer <token> }
Body: { spotId, pin }
Response: { valid: boolean, holdId, driverId }

GET /api/holds/active
Headers: { Authorization: Bearer <token> }
Response: { hold: Hold | null }

DELETE /api/holds/:holdId
Headers: { Authorization: Bearer <token> }
Response: { cancelled: boolean }
```

#### Sessions

```
POST /api/sessions/activate
Headers: { Authorization: Bearer <token> }
Body: { holdId, upiTransactionId }
Response: { sessionId, startTime }

POST /api/sessions/:sessionId/end
Headers: { Authorization: Bearer <token> }
Response: { endTime, duration, totalCost }

GET /api/sessions/active
Headers: { Authorization: Bearer <token> }
Response: { session: Session | null }

GET /api/sessions/host-active
Headers: { Authorization: Bearer <token> }
Response: { sessions: Session[] }

POST /api/sessions/:sessionId/report-overstay
Headers: { Authorization: Bearer <token> }
Response: { reported: boolean }

GET /api/sessions/history
Headers: { Authorization: Bearer <token> }
Response: { sessions: Session[] }
```

#### Payment

```
POST /api/payment/generate-upi-link
Headers: { Authorization: Bearer <token> }
Body: { hostUpiId, amount, note }
Response: { upiLink }

POST /api/payment/disputes
Headers: { Authorization: Bearer <token> }
Body: { sessionId, reason }
Response: { disputeId }
```

### Socket.io Events

#### Client → Server

```
'spot:watch' - { spotId }
'spot:unwatch' - { spotId }
'connection:authenticate' - { token }
```

#### Server → Client

```
'spot:status:changed' - { spotId, status, timestamp }
'hold:created' - { holdId, spotId, expiresAt }
'hold:expired' - { holdId, spotId }
'pin:verified' - { holdId, spotId }
'session:activated' - { sessionId, spotId, startTime }
'session:ended' - { sessionId, spotId, endTime }
'overstay:alert' - { sessionId, message, count }
'connection:restored' - { activeSession?, activeHold? }
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, several redundancies were identified:
- Requirements 2.3 and 4.5 both specify TTL expiration behavior (consolidated into Property 3)
- Requirements 1.1 and 15.4 both specify geospatial radius queries (consolidated into Property 1)
- Requirements 3.3 and 3.1 both address PIN uniqueness (consolidated into Property 4)
- Requirements 14.1, 14.2, 14.3, and 14.4 all address session state persistence (consolidated into Property 24)
- Requirements 13.1 and 13.2 both address device ID storage (consolidated into Property 28)

### Properties

**Property 1: Geospatial query completeness and correctness**
*For any* location coordinates and any set of spots, when searching within 500 meters, all returned spots must be within 500 meters of the location AND all spots within 500 meters must be included in the results.
**Validates: Requirements 1.1, 15.4**

**Property 2: Search result completeness**
*For any* spot returned in search results, the response must include address, distance, hourly rate, and host rating fields.
**Validates: Requirements 1.2**

**Property 3: Hold expiration and spot availability**
*For any* hold that expires without PIN verification, the associated spot must automatically transition to "available" status.
**Validates: Requirements 2.3, 4.5**

**Property 4: PIN uniqueness across active holds and sessions**
*For any* set of active holds and sessions, all generated PINs must be unique (no two active holds or sessions can have the same PIN).
**Validates: Requirements 3.1, 3.3**

**Property 5: Hold mutual exclusion per spot**
*For any* spot, when a hold exists for that spot, attempting to create another hold for the same spot must fail.
**Validates: Requirements 2.2**

**Property 6: Hold mutual exclusion per driver**
*For any* driver, when an active hold exists for that driver, attempting to create another hold must fail.
**Validates: Requirements 2.5**

**Property 7: Hold creation with correct expiration**
*For any* valid spot and driver, creating a hold must set the expiration time to exactly 15 minutes from creation time.
**Validates: Requirements 2.1**

**Property 8: Real-time spot status broadcast**
*For any* spot status change, all clients watching that spot must receive a socket event containing the new status within the event loop cycle.
**Validates: Requirements 1.4, 2.4**

**Property 9: Expired hold PIN invalidation**
*For any* hold that has expired, attempting to verify its PIN must fail with an error indicating the hold is no longer valid.
**Validates: Requirements 3.4**

**Property 10: PIN verification state transition**
*For any* valid PIN entered by a host, the associated hold must transition from "pending" to "verified" status.
**Validates: Requirements 4.2**

**Property 11: Invalid PIN rejection**
*For any* invalid PIN (non-existent or mismatched), the verification attempt must return an error and the hold status must remain unchanged.
**Validates: Requirements 4.3**

**Property 12: PIN verification notification**
*For any* successful PIN verification, the driver associated with that hold must receive a socket event notifying them of verification completion.
**Validates: Requirements 4.4**

**Property 13: UPI link generation correctness**
*For any* verified hold, the generated UPI Intent Link must contain the correct host UPI ID and the calculated amount based on hourly rate.
**Validates: Requirements 5.1**

**Property 14: Transaction ID persistence**
*For any* session activation with a UPI transaction ID, the system must store the transaction ID and it must be retrievable from the session record.
**Validates: Requirements 5.4**

**Property 15: Session activation from hold**
*For any* verified hold and valid transaction ID, activating the session must create a session record with status "active" and record the start time.
**Validates: Requirements 6.1, 6.4**

**Property 16: Session activation spot status update**
*For any* session activation, the associated spot status must change to "active" and a socket event must be emitted to all watchers.
**Validates: Requirements 6.2**

**Property 17: Session activation host notification**
*For any* session activation, the host associated with the spot must receive a socket event containing the session details.
**Validates: Requirements 6.3**

**Property 18: Active session retrieval**
*For any* driver with an active session, querying for their active session must return the correct session with all details including start time and spot information.
**Validates: Requirements 6.5**

**Property 19: Session end spot availability**
*For any* active session, when the session is ended, the associated spot must transition to "available" status and a socket event must be emitted.
**Validates: Requirements 7.2**

**Property 20: Session end time and duration calculation**
*For any* session that is ended, the system must record the end time and calculate duration as the difference between end time and start time.
**Validates: Requirements 7.3**

**Property 21: Session history completeness**
*For any* user (driver or host), their session history must include all completed sessions with start time, end time, and duration.
**Validates: Requirements 7.4**

**Property 22: Overstay notification delivery**
*For any* overstay report, the driver associated with the session must receive a socket event with the overstay alert.
**Validates: Requirements 8.1**

**Property 23: Overstay notification repetition**
*For any* session with overstay reported, if the session remains active, overstay notifications must be sent repeatedly (testable by verifying notification count increments).
**Validates: Requirements 8.2**

**Property 24: Session state persistence and recovery**
*For any* active session or hold, the state must be persisted in the database and retrievable after client reconnection, including all fields (PIN, status, timestamps).
**Validates: Requirements 12.2, 12.3, 14.1, 14.2, 14.3, 14.4**

**Property 25: Spot creation validation - photos required**
*For any* spot creation attempt without photos, the system must reject the request with a validation error.
**Validates: Requirements 9.1**

**Property 26: Spot creation validation - required fields**
*For any* spot creation attempt missing address, hourly rate, or availability schedule, the system must reject the request with a validation error.
**Validates: Requirements 9.2**

**Property 27: Spot creation validation - positive rate**
*For any* spot creation attempt with hourly rate ≤ 0, the system must reject the request with a validation error.
**Validates: Requirements 9.5**

**Property 28: Device ID storage and association**
*For any* user registration, the system must store the device ID and associate it with the user account (retrievable from user record).
**Validates: Requirements 9.3, 13.1, 13.2**

**Property 29: GeoJSON format compliance**
*For any* spot stored in the database, the location field must follow GeoJSON Point format with coordinates as [longitude, latitude].
**Validates: Requirements 9.4, 15.3**

**Property 30: Fraud report account suspension**
*For any* fraud report against a spot, the host's account status must change to "suspended".
**Validates: Requirements 10.1**

**Property 31: Suspended host spot unavailability**
*For any* host with suspended account status, all their spots must have status "suspended" or "unavailable".
**Validates: Requirements 10.2**

**Property 32: Banned device ID logging**
*For any* suspended account, the associated device ID must be added to the banned devices collection.
**Validates: Requirements 10.3**

**Property 33: Banned device registration rejection**
*For any* registration attempt with a device ID in the banned devices collection, the registration must fail with an error.
**Validates: Requirements 10.4**

**Property 34: Dispute record completeness**
*For any* payment dispute created, the dispute record must include the UPI transaction ID, session ID, driver ID, host ID, and timestamp.
**Validates: Requirements 11.1, 11.4**

**Property 35: Transaction ID audit trail**
*For any* session with a transaction ID, the transaction ID must be stored with a timestamp and be retrievable for audit purposes.
**Validates: Requirements 11.3**

**Property 36: Device ID authentication validation**
*For any* login attempt, if the device ID does not match the registered device ID for that user, the authentication must fail.
**Validates: Requirements 13.3**

**Property 37: OTP authentication success**
*For any* valid phone number and correct OTP, the authentication must succeed and return a valid JWT token.
**Validates: Requirements 13.4**

**Property 38: Banned device list persistence**
*For any* device ID added to the banned list, it must be retrievable from the banned devices collection.
**Validates: Requirements 13.5**

## Error Handling

### Error Categories

#### 1. Validation Errors (HTTP 400)

**Scenarios:**
- Missing required fields in spot creation (address, rate, photos, schedule)
- Invalid hourly rate (≤ 0)
- Invalid coordinates format
- Invalid phone number format
- Invalid PIN format (not 4 digits)
- Empty or whitespace-only transaction ID

**Response Format:**
```json
{
  "error": "VALIDATION_ERROR",
  "message": "Human-readable error message",
  "fields": {
    "fieldName": "Specific field error"
  }
}
```

#### 2. Authentication Errors (HTTP 401)

**Scenarios:**
- Invalid or expired JWT token
- Invalid OTP
- Device ID mismatch during login
- Banned device attempting registration

**Response Format:**
```json
{
  "error": "AUTHENTICATION_ERROR",
  "message": "Authentication failed",
  "reason": "INVALID_TOKEN | INVALID_OTP | DEVICE_MISMATCH | BANNED_DEVICE"
}
```

#### 3. Authorization Errors (HTTP 403)

**Scenarios:**
- Driver attempting to verify PIN for another driver's hold
- Host attempting to access another host's spots
- Suspended account attempting actions

**Response Format:**
```json
{
  "error": "AUTHORIZATION_ERROR",
  "message": "Not authorized to perform this action"
}
```

#### 4. Resource Not Found (HTTP 404)

**Scenarios:**
- Spot ID doesn't exist
- Hold ID doesn't exist
- Session ID doesn't exist
- User ID doesn't exist

**Response Format:**
```json
{
  "error": "NOT_FOUND",
  "message": "Resource not found",
  "resource": "spot | hold | session | user"
}
```

#### 5. Conflict Errors (HTTP 409)

**Scenarios:**
- Attempting to create hold when driver already has active hold
- Attempting to create hold when spot already has active hold
- Attempting to activate session when spot is not in verified state
- Attempting to register with existing phone number

**Response Format:**
```json
{
  "error": "CONFLICT",
  "message": "Resource conflict",
  "reason": "DRIVER_HAS_ACTIVE_HOLD | SPOT_ALREADY_HELD | INVALID_STATE | DUPLICATE_PHONE"
}
```

#### 6. Business Logic Errors (HTTP 422)

**Scenarios:**
- PIN verification failed (invalid PIN)
- Hold expired before PIN verification
- Session already ended
- Spot not available for booking

**Response Format:**
```json
{
  "error": "BUSINESS_LOGIC_ERROR",
  "message": "Operation cannot be completed",
  "reason": "INVALID_PIN | HOLD_EXPIRED | SESSION_ENDED | SPOT_UNAVAILABLE"
}
```

#### 7. Database Errors (HTTP 500)

**Scenarios:**
- MongoDB connection failure
- Query timeout
- Index creation failure
- TTL index not working

**Response Format:**
```json
{
  "error": "DATABASE_ERROR",
  "message": "Database operation failed",
  "retryable": true
}
```

**Handling Strategy:**
- Log full error details server-side
- Return generic message to client
- Implement exponential backoff for retries
- Alert monitoring system for persistent failures

#### 8. Socket Connection Errors

**Scenarios:**
- WebSocket connection lost
- Authentication failure on socket connection
- Room join failure

**Handling Strategy:**
- Client-side: Automatic reconnection with exponential backoff
- Server-side: Clean up stale connections after timeout
- Emit 'connection:restored' event with state recovery data
- Queue messages during disconnection and replay on reconnect

### Error Recovery Patterns

#### Hold Expiration Recovery
```
1. Hold expires due to timeout
2. TTL index removes hold document
3. Spot status automatically resets to "available"
4. Socket event broadcasts status change
5. Driver receives "hold:expired" notification
```

#### Payment Dispute Recovery
```
1. Driver or host raises dispute
2. System creates dispute record with all transaction details
3. Manual review process initiated
4. Host manually verifies UPI transaction ID
5. Dispute resolved with outcome recorded
```

#### Session State Recovery
```
1. Client browser reloads or reconnects
2. Client sends authentication token via socket
3. Server queries database for active holds/sessions
4. Server sends 'connection:restored' event with state
5. Client UI updates to reflect current state
```

#### Fraud Report Recovery
```
1. Driver reports fraudulent spot
2. Host account immediately suspended
3. All host spots marked unavailable
4. Device ID added to banned list
5. Active sessions for that host allowed to complete
6. Manual review process for appeal
```

## Testing Strategy

### Dual Testing Approach

The testing strategy employs both unit tests and property-based tests as complementary approaches:

- **Unit tests**: Verify specific examples, edge cases, error conditions, and integration points
- **Property-based tests**: Verify universal properties across randomized inputs (minimum 100 iterations per test)

Together, these approaches provide comprehensive coverage: unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide input space.

### Property-Based Testing Configuration

**Library Selection:**
- **JavaScript/TypeScript**: fast-check (recommended for Node.js/React)
- **Alternative**: jsverify

**Configuration:**
- Minimum 100 iterations per property test
- Each test must reference its design document property
- Tag format: `// Feature: spot-share-track, Property {number}: {property_text}`

**Example Property Test Structure:**

```typescript
import fc from 'fast-check';

// Feature: spot-share-track, Property 1: Geospatial query completeness and correctness
describe('Geospatial Query Properties', () => {
  it('should return all spots within 500m and only spots within 500m', () => {
    fc.assert(
      fc.property(
        fc.record({
          lat: fc.double({ min: -90, max: 90 }),
          lng: fc.double({ min: -180, max: 180 }),
        }),
        fc.array(fc.record({
          spotId: fc.uuid(),
          lat: fc.double({ min: -90, max: 90 }),
          lng: fc.double({ min: -180, max: 180 }),
          status: fc.constantFrom('available', 'held', 'active'),
        })),
        async (searchLocation, spots) => {
          // Setup: Insert spots into test database
          await setupTestSpots(spots);
          
          // Execute: Search within 500m
          const results = await spotService.searchSpots(searchLocation, 500);
          
          // Verify: All results within 500m
          for (const result of results) {
            const distance = calculateDistance(searchLocation, result.location);
            expect(distance).toBeLessThanOrEqual(500);
          }
          
          // Verify: All spots within 500m are included
          const spotsWithin500m = spots.filter(spot => {
            const distance = calculateDistance(searchLocation, { lat: spot.lat, lng: spot.lng });
            return distance <= 500 && spot.status === 'available';
          });
          
          expect(results.length).toBe(spotsWithin500m.length);
          
          // Cleanup
          await cleanupTestSpots();
        }
      ),
      { numRuns: 100 }
    );
  });
});
```

### Unit Testing Strategy

**Focus Areas:**

1. **Edge Cases**
   - Empty search results (no spots within radius)
   - Hold expiration at exactly 15 minutes
   - PIN verification with expired hold
   - Session activation with invalid transaction ID
   - Spot creation with empty photo array

2. **Error Conditions**
   - Invalid PIN format (non-numeric, wrong length)
   - Duplicate hold creation attempts
   - Authentication with banned device ID
   - Fraud report on non-existent spot
   - Session end on already-ended session

3. **Integration Points**
   - Socket event emission and reception
   - MongoDB TTL index behavior
   - JWT token generation and validation
   - Device fingerprint generation
   - UPI link format validation

4. **State Transitions**
   - Spot: available → held → active → available
   - Hold: pending → verified → expired
   - Session: active → completed
   - Account: active → suspended

**Example Unit Test:**

```typescript
describe('Hold Service - Edge Cases', () => {
  it('should reject hold creation when driver already has active hold', async () => {
    // Setup
    const driverId = 'driver-123';
    const spotId1 = 'spot-456';
    const spotId2 = 'spot-789';
    
    // Create first hold
    const hold1 = await holdService.createHold(driverId, spotId1);
    expect(hold1.holdId).toBeDefined();
    
    // Attempt second hold
    await expect(
      holdService.createHold(driverId, spotId2)
    ).rejects.toThrow('DRIVER_HAS_ACTIVE_HOLD');
  });
  
  it('should auto-expire hold after 15 minutes', async () => {
    // Setup
    const driverId = 'driver-123';
    const spotId = 'spot-456';
    
    // Create hold
    const hold = await holdService.createHold(driverId, spotId);
    
    // Fast-forward time by 15 minutes (using test clock)
    await advanceTime(15 * 60 * 1000);
    
    // Verify hold expired
    const spot = await spotService.getSpot(spotId);
    expect(spot.status).toBe('available');
    
    // Verify PIN no longer valid
    await expect(
      holdService.verifyPIN(spotId, hold.pin)
    ).rejects.toThrow('HOLD_EXPIRED');
  });
});
```

### Test Coverage Requirements

**Minimum Coverage Targets:**
- Line coverage: 80%
- Branch coverage: 75%
- Function coverage: 90%

**Critical Paths (100% coverage required):**
- Hold creation and expiration logic
- PIN generation and verification
- Session activation and state transitions
- Fraud reporting and account suspension
- Device ID validation and banning

### Integration Testing

**Socket.io Integration:**
- Test real WebSocket connections
- Verify event emission and reception
- Test reconnection logic
- Verify room-based broadcasting

**MongoDB Integration:**
- Test TTL index expiration behavior
- Test geospatial queries with real coordinates
- Test transaction handling for atomic operations
- Test index performance with large datasets

**End-to-End Workflow Tests:**
1. Complete "Verified Handshake" flow (Hold → PIN → Payment → Activation)
2. Fraud report flow (Report → Suspension → Ban)
3. Session lifecycle (Activation → Overstay → Completion)
4. State recovery flow (Disconnect → Reconnect → State Restore)

### Performance Testing

**Load Testing Scenarios:**
- 1000 concurrent socket connections
- 100 simultaneous spot searches
- 50 concurrent hold creations
- Real-time event broadcasting to 500+ clients

**Performance Targets:**
- Geospatial query: < 2 seconds
- Hold creation: < 500ms
- PIN verification: < 300ms
- Socket event delivery: < 100ms
- Session activation: < 1 second

### Test Data Management

**Generators for Property Tests:**
- Random coordinates (valid lat/lng ranges)
- Random spot data (address, rate, photos)
- Random user data (phone numbers, device IDs)
- Random PINs (4-digit strings)
- Random UPI transaction IDs (12-character alphanumeric)

**Test Database:**
- Use separate MongoDB instance for tests
- Clean up after each test
- Use transactions for atomic test setup/teardown
- Seed with realistic data for integration tests

### Continuous Integration

**CI Pipeline:**
1. Lint and format check
2. Unit tests (fast-check with 100 iterations)
3. Integration tests
4. Property-based tests (extended runs with 1000 iterations)
5. Coverage report generation
6. Performance benchmarks

**Pre-commit Hooks:**
- Run unit tests
- Run linter
- Check for console.log statements
- Verify no hardcoded credentials
