# Requirements Document: Spot Share Track

## Introduction

Spot Share Track is a peer-to-peer parking marketplace designed for urban India, enabling homeowners to rent unused driveways and gate spaces to drivers seeking parking. The system implements a high-integrity "Verified Handshake" workflow that ensures trust between hosts and drivers through manual verification steps, real-time state management, and direct UPI payments. The platform operates as a Progressive Web App (PWA) built with React, Node.js, Express, Socket.io, and MongoDB, without relying on AI or expensive third-party APIs.

## Glossary

- **System**: The Spot Share Track platform (web application, backend services, and database)
- **Driver**: A user seeking to rent a parking spot
- **Host**: A user offering their parking space for rent
- **Spot**: A parking space offered by a Host
- **Hold**: A temporary reservation state that prevents other Drivers from booking the same Spot
- **PIN**: A 4-digit verification code generated for the Driver to prove arrival at the Spot
- **Handshake**: The verification process where the Host enters the Driver's PIN to confirm presence
- **Session**: The active parking period from activation to completion
- **UPI_Transaction_ID**: The unique identifier from a UPI payment transaction
- **Socket_Connection**: A real-time bidirectional communication channel between client and server
- **TTL_Index**: MongoDB Time-To-Live index that automatically expires documents after a specified duration
- **Device_ID**: A unique identifier for a user's device used for fraud prevention
- **Geospatial_Query**: A database query that finds locations within a specified radius

## Requirements

### Requirement 1: Spot Discovery and Search

**User Story:** As a Driver, I want to search for available parking spots near my destination, so that I can find convenient parking options.

#### Acceptance Criteria

1. WHEN a Driver provides a location, THE System SHALL return all available Spots within 500 meters using Geospatial_Query
2. WHEN displaying search results, THE System SHALL show Spot address, distance, hourly rate, and Host rating
3. WHEN no Spots are available within 500 meters, THE System SHALL display a message indicating no results found
4. THE System SHALL update search results in real-time when Spot availability changes

### Requirement 2: Spot Reservation (Hold Phase)

**User Story:** As a Driver, I want to reserve a parking spot temporarily, so that I can secure it while traveling to the location.

#### Acceptance Criteria

1. WHEN a Driver selects an available Spot, THE System SHALL create a Hold with a 15-minute expiration using TTL_Index
2. WHEN a Hold is created, THE System SHALL prevent other Drivers from booking the same Spot
3. WHEN a Hold expires after 15 minutes without PIN verification, THE System SHALL automatically reset the Spot status to available
4. WHEN a Hold is created, THE System SHALL broadcast the Spot status change to all connected clients via Socket_Connection
5. THE System SHALL allow only one active Hold per Driver at any time

### Requirement 3: Arrival Verification (PIN Generation)

**User Story:** As a Driver, I want to receive a verification PIN when I reserve a spot, so that I can prove my arrival to the Host.

#### Acceptance Criteria

1. WHEN a Hold is created, THE System SHALL generate a unique 4-digit PIN for the Driver
2. THE System SHALL display the PIN prominently on the Driver's screen
3. THE System SHALL ensure the PIN is unique across all active Sessions
4. WHEN a Hold expires, THE System SHALL invalidate the associated PIN

### Requirement 4: Presence Confirmation (Handshake Phase)

**User Story:** As a Host, I want to verify a Driver's arrival by entering their PIN, so that I can confirm they are physically present at my parking spot.

#### Acceptance Criteria

1. WHEN a Host enters a 4-digit PIN, THE System SHALL validate it against active Holds for that Host's Spot
2. WHEN a valid PIN is entered, THE System SHALL transition the Hold to a verified state
3. WHEN an invalid PIN is entered, THE System SHALL display an error message and allow retry
4. WHEN a PIN is verified, THE System SHALL notify the Driver via Socket_Connection that verification is complete
5. IF a PIN is not entered within 15 minutes of Hold creation, THEN THE System SHALL expire the Hold and reset the Spot to available

### Requirement 5: Payment Processing

**User Story:** As a Driver, I want to pay the Host directly via UPI after arrival verification, so that I can complete the transaction securely.

#### Acceptance Criteria

1. WHEN a PIN is verified, THE System SHALL generate a UPI Intent Link with the Host's UPI ID and calculated amount
2. THE System SHALL display the UPI Intent Link to the Driver for payment
3. WHEN the Driver completes payment, THE System SHALL prompt the Driver to enter the last 4 digits of the UPI_Transaction_ID
4. THE System SHALL store the full UPI_Transaction_ID entered by the Driver for dispute resolution
5. THE System SHALL not validate the UPI_Transaction_ID automatically with payment gateways

### Requirement 6: Session Activation

**User Story:** As a Driver, I want to activate my parking session after payment, so that the Host knows I have paid and my parking time begins.

#### Acceptance Criteria

1. WHEN a Driver enters the last 4 digits of a UPI_Transaction_ID, THE System SHALL activate the Session
2. WHEN a Session is activated, THE System SHALL update the Spot status to "Active" in real-time via Socket_Connection
3. WHEN a Session is activated, THE System SHALL display the active Session on the Host's dashboard immediately
4. THE System SHALL record the Session start time when activated
5. THE System SHALL allow the Driver to view their active Session details including start time and Spot information

### Requirement 7: Session Monitoring and Completion

**User Story:** As a Host, I want to monitor active parking sessions, so that I can track when Drivers are using my spot.

#### Acceptance Criteria

1. WHEN a Session is active, THE System SHALL display real-time Session status on the Host's dashboard
2. WHEN a Driver ends a Session, THE System SHALL update the Spot status to "Available" via Socket_Connection
3. WHEN a Session ends, THE System SHALL record the end time and calculate total duration
4. THE System SHALL display Session history to both Driver and Host including start time, end time, and duration

### Requirement 8: Overstay Management

**User Story:** As a Host, I want to notify a Driver when they overstay their reserved time, so that I can reclaim my parking spot.

#### Acceptance Criteria

1. WHEN a Host triggers "Report Overstay", THE System SHALL send a high-priority notification to the Driver via Socket_Connection
2. WHEN an overstay notification is sent, THE System SHALL repeat the notification every 2 minutes until the Driver ends the Session
3. THE System SHALL deliver overstay notifications as vibrating alerts on the Driver's device
4. WHEN a Driver receives an overstay notification, THE System SHALL display the notification prominently with an option to end the Session

### Requirement 9: Host Onboarding and Spot Creation

**User Story:** As a Host, I want to list my parking space with verification, so that Drivers can trust my listing is legitimate.

#### Acceptance Criteria

1. WHEN a Host creates a Spot listing, THE System SHALL require upload of at least one photo of the parking space
2. WHEN creating a Spot, THE System SHALL require the Host to provide address, hourly rate, and availability schedule
3. THE System SHALL store the Host's Device_ID during onboarding
4. WHEN a Spot is created, THE System SHALL store geospatial coordinates for the address using MongoDB 2dsphere indexing
5. THE System SHALL validate that the hourly rate is a positive number

### Requirement 10: Fraud Prevention and Reporting

**User Story:** As a Driver, I want to report fraudulent parking spots, so that the platform maintains quality and trust.

#### Acceptance Criteria

1. WHEN a Driver reports a Spot as fraudulent, THE System SHALL immediately suspend the Host's account
2. WHEN a Host account is suspended, THE System SHALL mark all their Spots as unavailable
3. THE System SHALL log the Device_ID of suspended accounts to prevent re-registration
4. WHEN a user attempts to register with a Device_ID associated with a suspended account, THE System SHALL reject the registration
5. THE System SHALL provide a "Report Fraud" button visible to Drivers during the Hold and Session phases

### Requirement 11: Payment Dispute Resolution

**User Story:** As a Host, I want to verify payment details when disputes arise, so that I can confirm whether a Driver actually paid.

#### Acceptance Criteria

1. WHEN a payment dispute is raised, THE System SHALL display the UPI_Transaction_ID entered by the Driver
2. THE System SHALL allow the Host to manually confirm whether the UPI_Transaction_ID matches their received payment
3. THE System SHALL store all UPI_Transaction_IDs with timestamps for audit purposes
4. THE System SHALL maintain a dispute log linking Drivers, Hosts, Sessions, and UPI_Transaction_IDs

### Requirement 12: Real-Time State Synchronization

**User Story:** As a user, I want the app to reflect current parking spot status immediately, so that I have accurate information.

#### Acceptance Criteria

1. WHEN a Spot status changes, THE System SHALL broadcast the update to all connected clients within 1 second via Socket_Connection
2. WHEN a Driver's browser reloads during an active Session, THE System SHALL restore the Session state from the backend
3. WHEN a Host's browser reloads, THE System SHALL restore all active Sessions for their Spots from the backend
4. THE System SHALL maintain Socket_Connection for all authenticated users
5. WHEN a Socket_Connection is lost, THE System SHALL attempt to reconnect automatically

### Requirement 13: User Authentication and Device Tracking

**User Story:** As the System, I want to track user devices, so that I can prevent banned users from creating new accounts.

#### Acceptance Criteria

1. WHEN a user registers, THE System SHALL generate and store a unique Device_ID
2. THE System SHALL associate the Device_ID with the user account
3. WHEN a user logs in, THE System SHALL verify the Device_ID matches the registered device
4. THE System SHALL allow users to authenticate using phone number and OTP
5. THE System SHALL maintain a list of banned Device_IDs

### Requirement 14: Session State Persistence

**User Story:** As a Driver, I want my active parking session to persist if I close my browser, so that I don't lose my reservation.

#### Acceptance Criteria

1. WHEN a Driver has an active Session, THE System SHALL store the Session state in the database
2. WHEN a Driver reopens the app, THE System SHALL retrieve and display any active Session
3. WHEN a Host reopens the app, THE System SHALL retrieve and display all active Sessions for their Spots
4. THE System SHALL persist Hold state, PIN, Session status, and timestamps in the database

### Requirement 15: Geospatial Indexing and Performance

**User Story:** As the System, I want to efficiently query parking spots by location, so that search results are fast and accurate.

#### Acceptance Criteria

1. THE System SHALL use MongoDB 2dsphere indexes for all geospatial queries
2. WHEN performing a Geospatial_Query, THE System SHALL return results within 2 seconds
3. THE System SHALL store Spot coordinates in GeoJSON format
4. THE System SHALL support radius-based queries for Spot discovery
