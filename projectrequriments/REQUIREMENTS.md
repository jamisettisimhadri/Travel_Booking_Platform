# Travel Booking Platform --- Requirements
## 1. Project Overview

The **Travel Booking Platform** is a backend-focused travel reservation
system that will eventually support hotel, cab, and flight bookings.

For **Phase 1**, the project focuses on the **Hotel Booking Module**.
The platform will provide secure authentication, hotel/room search, room
booking, cancellation, and admin inventory management.

The most important engineering challenge in Phase 1 is ensuring that
**two users cannot successfully book the same room for overlapping
dates, even when booking requests arrive concurrently**.

------------------------------------------------------------------------

## 2. Problem Statement

Travelers need a single platform where they can search for
accommodation, check availability, make bookings, and manage their
reservations without relying on manual support.

Hotel operators/admins need to manage hotel and room inventory and
monitor bookings without accidentally allowing the same room to be
booked by multiple users.

The platform must therefore provide:

-   Secure user authentication and authorization
-   Fast hotel and room search
-   Date-based room availability
-   Reliable booking and cancellation
-   Consistent API error responses
-   Protection against concurrent double booking
-   Automated testing for critical functionality
-   Containerized deployment

Cab and flight booking capabilities are planned for later phases and are
**not part of Phase 1 implementation**.

------------------------------------------------------------------------

## 3. Users and Roles

### 3.1 Guest / Customer

A registered customer can:

-   Register an account
-   Log in
-   Receive a JWT access token
-   Search hotels
-   View hotel and room details
-   Check room availability
-   Create hotel bookings
-   View their own bookings
-   Cancel their own bookings
-   Receive consistent API error responses

### 3.2 Admin

An administrator can:

-   Create hotels
-   Update hotel information
-   Deactivate hotels
-   Create and manage rooms
-   Update room information
-   Deactivate rooms
-   View all bookings
-   Monitor booking status

Refund management will be introduced with the payment module in a later
phase.

### 3.3 External Payment Gateway

The payment gateway will act as an external webhook source in a later
phase.

For Phase 1, **real payment processing and payment webhooks are out of
scope**.

------------------------------------------------------------------------

## 4. Phase 1 Functional Requirements

### FR1 --- User Registration and Authentication

The system shall allow users to:

-   Register using required account details
-   Log in using valid credentials
-   Receive a JWT access token after successful authentication
-   Access protected endpoints only with valid authentication

Passwords must never be stored as plain text.

------------------------------------------------------------------------

### FR2 --- Hotel and Room Management

Admins shall be able to:

-   Create hotels
-   Update hotels
-   Deactivate hotels
-   Create rooms under hotels
-   Update room information
-   Deactivate rooms

A deactivated hotel or room must not be offered for new bookings.

------------------------------------------------------------------------

### FR3 --- Hotel Search

Authenticated users shall be able to search hotels using:

-   City
-   Check-in date
-   Check-out date
-   Number of guests
-   Price range

Search results shall support:

-   Pagination
-   Sorting
-   Availability filtering

The initial performance target is **less than 300 ms for hotel search on
a dataset of approximately 10,000 hotels**, measured under a defined
test environment.

------------------------------------------------------------------------

### FR4 --- Hotel and Room Details

Users shall be able to view:

-   Hotel details
-   Available room types
-   Room capacity
-   Room price
-   Room status
-   Basic hotel information

------------------------------------------------------------------------

### FR5 --- Hotel Booking

Users shall be able to create a booking for an available room.

A booking must contain at least:

-   User
-   Room
-   Check-in date
-   Check-out date
-   Number of guests
-   Booking status
-   Creation timestamp

The system must reject overlapping bookings for the same room.

#### Critical Requirement: Concurrent Booking Protection

Double booking must be prevented even when multiple users attempt to
book the same room at nearly the same time.

The implementation must use a database-level
transaction/concurrency-control strategy rather than relying only on
application-level availability checks.

This is a **highest-priority technical requirement** for Phase 1.

------------------------------------------------------------------------

### FR6 --- Booking Management

Customers shall be able to:

-   View their own bookings
-   View booking details
-   Cancel eligible bookings

Admins shall be able to:

-   View all bookings
-   View booking status
-   Monitor booking activity

Cancellation rules will be defined during the hotel module design in
Session 2.

------------------------------------------------------------------------

### FR7 --- Consistent API Errors

Every API error shall follow a consistent response structure containing:

-   Machine-readable error code
-   Human-readable message
-   Appropriate HTTP status code

Example structure:

``` json
{
  "code": "ROOM_NOT_AVAILABLE",
  "message": "The selected room is no longer available for the requested dates."
}
```

The exact error schema will be finalized during API design.

------------------------------------------------------------------------

## 5. Phase 1 Non-Functional Requirements

### NFR1 --- Concurrency and Data Integrity

The system must prevent double booking under concurrent booking
requests.

This requirement takes priority over simple application-level
availability checks.

### NFR2 --- Security

-   Passwords must be securely hashed.
-   JWT authentication must protect private endpoints.
-   Users must only access their own booking information.
-   Admin endpoints must require admin authorization.
-   Sensitive information must not be exposed in API errors.

### NFR3 --- Performance

Hotel search should target a response time of:

**\< 300 ms**

for approximately **10,000 hotels**, under a defined local/test
environment.

Performance will be measured later using realistic seed data and API
testing.

### NFR4 --- Automated Testing

Critical endpoints and business rules must have automated tests.

Testing should cover:

-   Authentication
-   Authorization
-   Hotel/room CRUD
-   Hotel search
-   Booking creation
-   Overlapping booking rejection
-   Concurrent booking attempts
-   Cancellation
-   Error response format

### NFR5 --- Containerization

The application must run using Docker with a simple startup command.

The final containerized setup should include the application and
database dependencies required by the project.

### NFR6 --- API Design

The backend should expose RESTful APIs with:

-   Clear resource-based endpoints
-   Appropriate HTTP methods
-   Proper status codes
-   Consistent JSON responses
-   Pagination for list endpoints

------------------------------------------------------------------------

## 6. Assumptions and Constraints

-   Version 1 uses **INR** as the single currency.
-   No real hotel, airline, or cab provider API will be integrated
    initially.
-   Hotel and room data will be seeded or managed by admins.
-   Database timestamps will use **UTC**.
-   The UI may display dates/times according to the user's local
    timezone.
-   Phase 1 does not include real payment processing.
-   Razorpay test-mode integration is reserved for the later payment
    phase.
-   Cab and flight modules are deferred to later phases.
-   The initial system is designed as a modular backend so additional
    travel modules can be added later.

------------------------------------------------------------------------

## 7. Out of Scope for Phase 1

The following features will **not** be implemented in Phase 1:

### Cab Module

-   Cab/vehicle inventory
-   Driver management
-   Cab search
-   Cab booking

### Flight Module

-   Flight schedules
-   Seat inventory
-   Flight search
-   Flight booking

### Payment Module

-   Razorpay integration
-   Payment order creation
-   Payment verification
-   Payment webhooks
-   Refund processing

### AI / RAG Module

-   Policy chatbot
-   Document ingestion
-   Embeddings
-   Vector database
-   Retrieval-Augmented Generation

### Risk / Pricing Module

-   Dynamic pricing
-   Booking risk scoring
-   Fraud detection
-   Recommendation/risk models

------------------------------------------------------------------------

## 8. Planned Future Phases

### Phase 2 --- Extended Travel Modules

Potential additions:

-   Cab booking
-   Flight booking
-   Payment integration
-   Razorpay test-mode payment flow
-   Payment webhook handling
-   Refund processing

### Phase 3 --- Pricing and Risk

Potential additions:

-   Dynamic pricing
-   Booking risk prediction
-   Fraud/risk detection
-   Analytics

### Phase 4 --- AI Policy Assistant

Potential additions:

-   RAG-based travel policy chatbot
-   Policy document ingestion
-   Vector search
-   Context-aware answers
-   Booking-policy assistance

------------------------------------------------------------------------

## 9. Technology Direction

The project will be aligned with the existing technical profile and
backend goals.

### Backend

-   Python
-   FastAPI
-   REST APIs
-   JWT authentication

### Database

-   Mysql 8.1
-   Relational database design
-   Transactions and constraints for booking integrity

### Frontend

-   HTML
-   CSS
-   JavaScript

A React frontend can be introduced if required by the final project
scope.

### Testing

-   Pytest
-   API testing with Postman

### DevOps / Tools

-   Docker
-   Git
-   GitHub

------------------------------------------------------------------------

## 10. Critical Requirements Priority

The Phase 1 requirements are prioritized as follows:

  ----------------------------------------------------------------------------
  Priority                Requirement             Reason
  ----------------------- ----------------------- ----------------------------
  P0                      Prevent concurrent      Core system integrity
                          double booking          requirement

  P0                      JWT authentication and  Security foundation
                          authorization           

  P0                      Consistent API errors   Reliable API contract

  P1                      Hotel/room management   Required inventory
                                                  functionality

  P1                      Hotel search and        Core customer functionality
                          availability            

  P1                      Booking and             Core reservation
                          cancellation            functionality

  P1                      Automated tests         Ensures reliability

  P2                      Search performance      Important optimization
                                                  target

  P2                      Docker deployment       Deployment/reproducibility
  ----------------------------------------------------------------------------

------------------------------------------------------------------------

## 11. Definition of Done --- Session 1

Session 1 is complete when:

-   The project problem can be explained in approximately two minutes.
-   All Phase 1 user roles are defined.
-   Customer and admin responsibilities are clear.
-   Phase 1 functional requirements are documented.
-   Non-functional requirements are documented.
-   Double-booking prevention is identified as the highest-priority
    technical requirement.
-   Authentication and authorization requirements are defined.
-   The API error format requirement is defined.
-   Phase 1 and future-phase boundaries are clear.
-   This `REQUIREMENTS.md` file is committed to the Git repository.

------------------------------------------------------------------------

## 12. Session 2 Preparation

Session 2 will define the **Hotel Module architecture and API
contract**, including:

-   Module boundaries
-   Hotel entities
-   Room entities
-   Booking entities
-   Database relationships
-   Booking status
-   Availability rules
-   Cancellation rules
-   Endpoint list
-   Request/response structures
-   Authentication and authorization boundaries
-   Strategy for preventing concurrent double booking

**Phase 1 decision:** Focus on the **Hotel Booking Module**. Defer cab
and flight implementation until later phases so the project can first
demonstrate a robust, transaction-safe booking system.
