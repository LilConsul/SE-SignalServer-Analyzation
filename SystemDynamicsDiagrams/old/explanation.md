# 1 - Message Sending and Delivery Sequence

## 📱 Phase 1: Message Initiation & Authentication

### Step 1: Client Sends Message

- 📱 **SignalServiceMessageSender (C1)** creates an encrypted message
- The message is wrapped in a "sealed sender envelope" using Signal's **X3DH + Double Ratchet** encryption
- C1 sends `POST /v1/messages/{uuid}` with `Authorization: Basic {base64}` header

### Step 2: Load Balancing

- ⚖️ **HAProxy/ELB** receives the request
- Uses consistent hashing to route the request to the appropriate backend server
- This ensures the same user typically hits the same server for better caching

### Step 3: Authentication

- 🌐 **MessageResource.java** receives the routed request
- Calls 🔐 **AccountAuthenticator** to validate credentials
- AccountAuthenticator uses BCrypt hashing with timing-safe comparison (prevents timing attacks)
- Returns `AuthenticatedDevice{accountId, deviceId}` if valid

## 🔧 Phase 2: Message Processing & Validation

### Step 4: Message Validation

- 📤 **MessageSender.java** receives the authenticated request
- Validates the protobuf envelope structure
- Ensures message content doesn't exceed 256KB limit
- Extracts timestamp and urgency flags

## ⚡ Phase 3: Parallel Operations (This is Key!)

### Step 5A: Check Recipient Status (Redis Path)

- MessageSender queries 🔴 **Redis Cluster**
- Uses command: `SISMEMBER online_devices {account:device}`
- Redis maintains a SET of online devices with 30-second TTL (Time To Live)
- Returns Boolean: true if device is online, false if offline

### Step 5B: Store Message (DynamoDB Path - Happens Simultaneously)

- MessageSender stores message in 🗃️ **DynamoDB Messages** table
- Uses composite key structure:
    - **Primary Key (PK):** `{accountId}#{deviceId}`
    - **Sort Key (SK):** `timestamp#{messageId}`
- Includes Global Secondary Index on timestamp for efficient cleanup later
- Consumes 1 Write Capacity Unit (WCU)

## 🎯 Phase 4: Smart Delivery Decision

### Step 6A: If Recipient is OFFLINE (Redis returned false)

- 🔔 **PushNotificationManager** is triggered
- Rate limiting: maximum 100 notifications per minute per device
- For iOS: Uses APNS HTTP/2 with JWT authentication
- For Android: Uses FCM with OAuth2 + exponential backoff for retries
- Returns `DeliveryReceipt{messageId, deliveredAt}`

### Step 6B: If Recipient is ONLINE (Redis returned true)

- Message sent directly to 📱 **SignalServiceMessageReceiver (C2)** via WebSocket
- Uses WebSocket BINARY frame containing `MessageProtos.Envelope`
- WebSocket connection is persistent with 55-second heartbeat to detect disconnections
- C2 immediately sends back `ACK frame{messageId}` to confirm receipt

## 📤 Phase 5: Response to Sender

### Step 7: Response Chain

- MessageSender returns `SendMessageResponse{needsSync: boolean}` to MessageResource
- MessageResource sends `200 OK` with timing headers to HAProxy
- HAProxy forwards response with request latency metrics to C1

## 📥 Phase 6: Message Synchronization (For Offline/Multi-Device Scenarios)

### Step 8: WebSocket Connection Establishment

- 📱 **SignalServiceMessageReceiver (C2)** establishes WebSocket connection
- Uses `Sec-WebSocket-Protocol: signalservice` header
- Connection secured with TLS 1.3 + certificate pinning

### Step 9: Message Queue Retrieval

- C2 queries DynamoDB Messages table through MessageResource
- Uses `FilterExpression: timestamp > lastSync`
- Retrieves up to 100 messages with `ScanIndexForward: true` (chronological order)
- Uses eventually consistent reads with auto-scaling on Read Capacity Units

### Step 10: Batch Delivery

- MessageResource sends messages via WebSocket BINARY frames
- Batches limited to maximum 10MB or 1000 messages
- Implements back-pressure handling to prevent overwhelming the client
- Messages delivered in chronological order

### Step 11: Acknowledgment & Cleanup

- C2 sends `ACK batch{lastMessageId}` after processing
- System deletes acknowledged messages from DynamoDB
- Enforces 30-day maximum retention TTL for unacknowledged messages

---

# 2.1 - Phone Number Verification

## 🚀 PHASE 1: Verification Session Setup

### Step 1: Client Initiates Verification Session

- 📱 **Client App** sends `POST /v1/verification/session`
- **Request Body:** `{phoneNumber, pushToken, captcha}`
- **Rate Limiting:** 5 requests per hour per IP address (prevents abuse)

### Step 2: Load Balancing & Routing

- ⚖️ **Load Balancer** receives the request
- Routes to the appropriate API gateway instance
- Ensures even distribution of verification requests

### Step 3: Input Validation

- 🌐 **HTTP API Gateway** forwards to 🎯 **VerificationController**
- **VerificationController performs validation:**
    - 📋 Validates phone number format (E.164 standard)
    - 🌍 Checks against country code allowlist
    - 🚫 Validates against blocked phone number list

### Step 4: Registration Service Integration

- **VerificationController** calls 🎯 **RegistrationServiceClient**
- **gRPC call:** `createRegistrationSession(phoneNumber, sourceHost)`
- **Configuration:**
    - ⏱️ Timeout: 15 seconds
    - 🔄 Retry attempts: 3 maximum
- **Returns:** `RegistrationServiceSession{sessionId, expiration: 10min}`

### Step 5: Session Caching

- Session metadata stored in 🔴 **Redis Cache**
- **Cache Key:** `session:{sessionId}`
- **TTL (Time To Live):** 10 minutes
- **Response:** `VerificationSessionResponse{sessionId, requestedInfo[CAPTCHA]}`

## 📞 PHASE 2: Verification Code Delivery

### Step 6: Code Request

- 📱 **Client** sends `PATCH /v1/verification/session/{sessionId}`
- **Request Body:** `{transport: "sms", captchaResponse}`
- **Transport Options:** 📱 SMS or ☎️ Voice call

### Step 7: Session & Rate Limit Validation

- **VerificationController** validates:
    - 🔍 Session exists and hasn't expired (Redis lookup)
    - 🔐 CAPTCHA response is valid
    - 🚦 Rate limits: 1 request per minute per phone number

### Step 8: Code Generation & Delivery

- **RegistrationServiceClient** calls `sendVerificationCode(sessionId, SMS, clientType)`
- **SMS Provider Integration:**
    - 📤 Forwards to Twilio or other SMS provider
    - **Message Format:** "Your Signal code: 123456"
    - **Code Properties:**
        - ⏱️ Valid for: 10 minutes
        - 🔄 Maximum attempts: 3

### Step 9: Session State Update

- 🔴 **Redis Cache** updates session state:
    - 📊 `{codesSent++, lastCodeSent: timestamp}`
- **Response:** `VerificationSessionResponse{transport: "sms", nextCodeIn: 60s}`

## ✅ PHASE 3: Code Verification

### Step 10: Code Submission

- 📱 **Client** sends `PUT /v1/verification/session/{sessionId}/code`
- **Request Body:** `{verificationCode: "123456"}`

### Step 11: Code Validation Process

- **VerificationController** retrieves session from Redis
- **RegistrationServiceClient** calls `checkVerificationCode(sessionId, code)`
- **Validation checks:**
    - 🔍 Code matches the one that was sent
    - ⏱️ Code hasn't expired (10-minute window)
    - 🔢 User hasn't exceeded 3 attempts

### Step 12: Verification Results (Three Possible Outcomes)

#### ✅ **Scenario A: Code Valid**

- RSC returns: RegistrationServiceSession(verified: true)
- Redis updated: {verified: true, verifiedAt: timestamp}
- Response: VerificationComplete{sessionId, phoneNumber}
- HTTP Status: 200 OK

#### ❌ **Scenario B: Code Invalid**

- RSC returns: InvalidCodeException(attemptsRemaining: 2)
- HTTP Status: 422 Unprocessable Entity
- User can try again with remaining attempts

#### 🚫 **Scenario C: Too Many Failed Attempts**

- RSC returns: RateLimitedException
- Session blacklisted for 5 minutes in Redis
- HTTP Status: 429 Too Many Requests
- User must wait before requesting new code

---

# 2.2 - Account Registration

## 🚀 PHASE 1: Registration Request & Validation

### Step 1: Client Submits Registration Request

- 📱 **Client App** sends `POST /v1/registration`
- **Request Payload includes:**
    - 🆔 **sessionId:** Verified phone verification session ID
    - 👤 **accountAttributes:** User profile settings and preferences
    - 🔐 **identityKeys:** Cryptographic keypairs for ACI/PNI identities
    - 📱 **deviceCapabilities:** What features this device supports

### Step 2: Load Balancing & Routing

- ⚖️ **Load Balancer** receives the registration request
- Routes to available 🌐 **HTTP API Gateway** instance
- Forwards to 📝 **RegistrationController** for processing

### Step 3: Phone Verification Token Validation

- **RegistrationController** calls 🎫 **PhoneVerificationTokenManager**
- **Validation Process:**
    - 🔍 Validates session exists in the system
    - ✅ Checks verification status (must be verified=true)
    - ⏱️ Verifies session hasn't expired (16-second timeout)
- **Result:** Phone number ownership confirmed ✅

## 🔍 PHASE 2: Existing Account Check & Registration Lock

### Step 4: Existing Account Lookup

- **RegistrationController** calls 👥 **AccountsManager**
- **AccountsManager** queries 📊 **DynamoDB**:
    - **Query:** `accounts` table using `phoneNumberIndex` GSI
    - **Key:** `{phoneNumber}`
- **Returns:** `Optional<Account>` - existing account if found

### Step 5: Registration Lock Security Flow (If Account Exists)

When an existing account is found with registration lock enabled:

#### Step 5A: Rate Limit Check

- 🔒 **RegistrationLockVerificationManager** validates attempt limits
- **Rate Limiting:** 1 attempt per 5 minutes per account
- **Cache Check:** Retrieves attempt history from Redis

#### Step 5B: PIN Validation (If PIN Provided)

**✅ PIN Valid Scenario:**

```
- Validates PIN against stored Argon2 hash with account-specific salt
- Clears failed attempts from cache (Key: reglock:{uuid})
- Returns: Registration lock verified ✅
```

**❌ PIN Invalid Scenario:**

```
- Records failed attempt with timestamp
- Calculates idle days since account last seen
- Increments attempt counter in Redis (TTL: 5 minutes)
- Returns: RateLimitExceededException (HTTP 423)
- Response: {timeRemaining, attemptsLeft, idleDays}
```

#### Step 5C: 7-Day Bypass (Alternative Path)

- **Dormant Account Check:** If account idle > 7 days
- **Action:** Allow registration bypass without PIN
- **Rationale:** Account considered abandoned

## 👤 PHASE 3: Account Creation & Key Management

### Step 6: Account Identity Generation

**AccountsManager** creates new account identities:

- 🆔 **ACI UUID:** Account Identity (primary identifier)
- 📱 **PNI UUID:** Phone Number Identity (phone-linked ID)
- 🔐 **Device Creation:** Primary device (ID: 1) with auth token and salt
- 📊 **Metrics Initialization:** Account usage tracking setup

### Step 7: Parallel Storage Operations

Three operations happen simultaneously for performance:

#### Step 7A: Account Storage (DynamoDB)

```
- Table: accounts
- Partition Key: ACI
- Attributes: {
    phoneNumber,
    pni,
    devices[],
    createdAt,
    badges[],
    capabilities
  }
```

#### Step 7B: Key Storage (DynamoDB)

```
- Table: keys
- Keys: ACI + PNI identity keypairs
- Metadata: {
    algorithm,
    created,
    deviceId
  }
```

#### Step 7C: Cache Update (Redis)

```
- Key: account:{ACI}
- Value: Full account object
- TTL: 3600s (1 hour)
- Purpose: Fast account lookups
```

## 🎉 PHASE 4: Registration Completion

### Step 8: Success Response Generation

- **RegistrationController** receives successful account creation
- **Response Payload:**
    - 🆔 **uuid:** Account UUID (ACI)
    - 📱 **pni:** Phone Number Identity (PNI)
    - 📱 **deviceId:** Primary device ID (always 1)
    - 💾 **storageCapable:** Storage capability flag (true)

### Step 9: Response Chain

- **API Gateway** returns `HTTP 201 Created`
- **Load Balancer** forwards success response
- **Client App** receives registration completion confirmation
- **Status:** 🎉 Account registration complete! Ready for secure messaging

---

# 2.3 - Device Authentication & Session Management

## 🔐 PHASE 1: Device Authentication Setup

### Step 1: Initial Device Authentication Request

- 📱 **Client Device** sends `PUT /v1/accounts/attributes`
- **Authorization Header:** `Basic {number}:{password}` (Base64 encoded)
- **Purpose:** First-time device setup after registration
- **Request Payload includes:**
    - 📱 Device capabilities (GCM, APNS, WebSocket support)
    - 🔔 Push token registration
    - ⚙️ Client configuration and preferences

### Step 2: Load Balancing & Request Routing

- ⚖️ **Load Balancer** receives the authenticated request
- Routes to available 🌐 **HTTP API Gateway** instance
- Forwards to 🔐 **AuthenticationFilter** for credential validation

### Step 3: Basic Authentication Parsing

- **AuthenticationFilter** performs credential extraction:
    - 📋 Parses Basic Auth header
    - 📱 Extracts `{phoneNumber, authToken}` components
    - 🔍 Validates format and encoding integrity

### Step 4: Account Lookup & Verification

**Parallel Cache and Database Strategy:**

#### Step 4A: Cache Lookup (Primary Path)

- 👥 **AccountsManager** queries 🔴 **Redis Cache**
- **Cache Key:** `account:{phoneNumber}:auth`
- **TTL Check:** Validates 3600-second (1 hour) expiration
- **Result:** Cached account data or cache miss

#### Step 4B: Database Fallback (If Cache Miss)

- **AccountsManager** queries 📊 **DynamoDB**:
    - **Table:** `accounts`
    - **Index:** `phoneNumberIndex` GSI
    - **Filter:** `authToken` match validation
- **Post-Query:** Updates cache with account data (TTL: 1 hour)

### Step 5: Authentication Validation Result

- **AuthenticationFilter** receives `AuthenticatedAccount{uuid, deviceId, capabilities}`
- **API Gateway** receives authentication confirmation
- **Principal Created:** `{account, device}` for request context

## 📱 PHASE 2: Device Registration & Session Creation

### Step 6: Device Attribute Updates

- **API Gateway** calls 📱 **DeviceManager**
- **Device State Management includes:**
    - 🔄 Update device capabilities (GCM, APNS, WebSocket support)
    - 🔔 Update push tokens for notifications
    - ⏰ Set `lastSeen` timestamp for activity tracking

### Step 7: Session Creation Process

- **DeviceManager** calls 🎫 **SessionManager**
- **Session Lifecycle Management:**
    - 🆔 Generate unique session ID
    - ⏰ Set expiration (30 days from creation)
    - 🔐 Create cryptographically secure session token
    - 📊 Initialize session metrics and tracking

### Step 8: Parallel Storage Operations

Three operations happen simultaneously for performance:

#### Step 8A: Session Storage (Redis)

```
- Key: session:{sessionId}
- Value: {
    accountId,
    deviceId,
    created,
    expires,
    scope[]
  }
- TTL: 30 days
```

#### Step 8B: Device Update (DynamoDB)

```
- Table: accounts
- Key: {ACI}
- Update: devices[deviceId].{
    lastSeen,
    pushTokens,
    capabilities
  }
```

### Step 9: Session Response

- **SessionManager** returns `SessionInfo{sessionId, token, expires}`
- **DeviceManager** responds with `DeviceUpdateResponse{capabilities, sessionInfo}`
- **Client** receives session credentials for future API calls

## 🔄 PHASE 3: Ongoing Authentication & Session Validation

### Step 10: Subsequent API Request Authentication

For all future API calls, clients use Bearer token authentication:

- 📱 **Client** sends any API request with `Authorization: Bearer {sessionToken}`
- ⚖️ **Load Balancer** routes to **API Gateway**
- **AuthenticationFilter** validates session token

### Step 11: Session Validation Process

#### Step 11A: Session Lookup

- **AuthenticationFilter** queries 🔴 **Redis Cache**
- **Lookup:** `session:{tokenHash}` with expiration check
- **Validation:** Confirms session exists and hasn't expired

#### Step 11B: Account Verification

- **AuthenticationFilter** calls **AccountsManager**
- **Account Lookup:** Cached account details retrieval
- **Verification:** Confirms account is still active and valid

### Step 12: Session Validation Outcomes

#### ✅ **Valid Session Scenario:**

```
- Cache returns: Session data{accountId, deviceId, scope[]}
- Account verification: Success
- Principal created: {account, device, session}
- Parallel operations:
  - Session TTL extension based on usage
  - Activity tracking: INCR activity:{accountId}:{date}
```

#### ❌ **Invalid/Expired Session Scenario:**

```
- Cache returns: Session not found or expired
- Response: HTTP 401 Unauthorized
- Header: WWW-Authenticate: Bearer
- Client action: Re-authentication required
```

### Step 13: Request Processing

- **API Gateway** processes authenticated request with full context
- **Response:** Successful API operation completion
- **Metrics:** Request latency and success rate tracking

## 🔄 PHASE 4: Session Cleanup & Security

### Step 14: Background Session Management

**Periodic Cleanup (Every 6 Hours):**

#### Step 14A: Expired Session Cleanup

- **SessionManager** scans Redis for expired sessions
- **Cleanup Process:**
    - 🔍 `SCAN` expired sessions where TTL < current time
    - 🗑️ Remove session references from device records
    - 🧹 Clean up orphaned session data

#### Step 14B: Metrics Collection

- **SessionManager** collects session analytics:
    - 📊 Active sessions count
    - 📱 Device type distribution
    - 📈 Usage patterns and trends

### Step 15: Security Event Detection

**Automated Security Monitoring:**

#### Step 15A: Suspicious Activity Detection

- **SessionManager** monitors for anomalies:
    - 🔍 Multiple devices from different locations
    - 📱 Unusual access patterns
    - 🌍 Geographical anomalies

#### Step 15B: Security Response

- **AccountsManager** triggers security review:
    - 🚨 `triggerSecurityReview(accountId, event)`
    - 🔐 Revoke suspicious sessions automatically
    - 📧 Send security notification to user
    - ✅ Complete security review process

---

# 3 -System Infrastructure and Service Management

## 🚀 PHASE 1: System Initialization

### Step 1: Application Launch

- 🚀 **Main Application** initiates startup process
- Calls **WhisperServerService** with `run(configuration, environment)`
- **Purpose:** Begin the complete Signal server initialization sequence

### Step 2: Configuration Loading

- 🌐 **WhisperServerService** calls ⚙️ **Configuration Manager**
- **Configuration Process:**
    - 📂 Load configuration files from filesystem
    - 📖 Parse YAML configuration files
    - ✅ Validate all configuration parameters for correctness
- **Result:** ✅ Configuration loaded and validated

### Step 3: Environment Setup

- **WhisperServerService** initializes 🔧 **Environment Manager**
- **Environment Manager** creates ♻️ **Lifecycle Manager**
- **Purpose:** Set up application runtime environment and lifecycle management
- **Result:** ✅ Environment initialized and ready

## 🔗 PHASE 2: External Dependencies Setup

### Step 4: Redis Cluster Connections (Parallel Operation)

**Multiple Redis clusters are connected simultaneously:**

- 🔴 **Redis Clusters** initialization includes:
    - 💬 **Messages cluster:** Message queuing and temporary storage
    - 🔔 **Push scheduler cluster:** Push notification scheduling
    - 🚦 **Rate limiters cluster:** API rate limiting data
    - 📡 **Pubsub cluster:** Real-time message broadcasting

### Step 5: DynamoDB Connections (Parallel Operation)

**DynamoDB connections established simultaneously:**

- 📊 **DynamoDB** configuration includes:
    - 👥 **Account tables:** User account and device information
    - 🔑 **Keys tables:** Cryptographic key storage
    - 💬 **Messages tables:** Persistent message storage

**Result:** ✅ All external data stores connected and ready

## ⚡ PHASE 3: Thread Pool & Executor Setup

### Step 6: Executor Services Creation

- **WhisperServerService** calls ⚡ **Executor Services**
- **Thread Pool Configuration:**
    - 🌐 **Main thread pool:** HTTP request handling
    - 🔄 **Async thread pool:** Background processing tasks
    - ⏰ **Scheduled executor:** Periodic maintenance tasks
    - 🔌 **WebSocket executor:** Real-time connection handling
    - 📬 **Message delivery executor:** Message routing and delivery
    - 🔔 **Push notification executor:** Push notification processing

**Result:** ✅ All executor services ready for concurrent operations

## 🎯 PHASE 4: Business Services Initialization

### Step 7: Core Services Creation (Parallel Operation)

**Core business services initialized simultaneously:**

- 🎯 **Business Services** creation includes:
    - 👥 **AccountsManager:** User account management
    - 💬 **MessagesManager:** Message processing and routing
    - 🔑 **KeysManager:** Cryptographic key management
    - 🔔 **PushNotificationManager:** Push notification handling
    - 📤 **MessageSender:** Message delivery coordination
    - 🔐 **AuthenticationServices:** User authentication and authorization

### Step 8: Dynamic Configuration Setup (Parallel Operation)

- 🔄 **DynamicConfiguration** initialization:
    - 🔌 Connect to configuration backend
    - 🏁 Load feature flags and runtime parameters
    - 🔄 Setup configuration polling for real-time updates

**Result:** ✅ All business services and dynamic configuration active

## 🎮 PHASE 5: API Controllers & Server Setup

### Step 9: REST Controllers Registration

- **WhisperServerService** registers 🎮 **Controllers**
- **API Controller Registration:**
    - 💬 **MessageController:** Message sending/receiving endpoints
    - 👥 **AccountController:** Account management endpoints
    - 🔑 **KeysController:** Key exchange and management endpoints
    - 👤 **ProfileController:** User profile management endpoints
    - 📱 **DeviceController:** Device registration and management endpoints

### Step 10: Server Infrastructure Configuration

- 🖥️ **Server Infrastructure** setup includes:
    - 🌐 **HTTP server (Jetty):** REST API endpoint handling
    - 🔌 **WebSocket server:** Real-time bidirectional communication
    - ⚡ **gRPC servers:** High-performance service-to-service communication
    - 🔐 **Authentication filters:** Request authentication middleware
    - 🔍 **Request filters:** Request validation and preprocessing

**Result:** ✅ All servers configured and ready to accept requests

## 📊 PHASE 6: Monitoring & Health Checks

### Step 11: Metrics System Initialization (Parallel Operation)

- 📈 **Metrics System** setup:
    - 📊 Setup Micrometer registry for metrics collection
    - 📤 Configure metric exporters (Prometheus, CloudWatch, etc.)
    - 📝 Register custom business metrics

### Step 12: Health Monitoring Setup (Parallel Operation)

- 💚 **Health Checks** configuration:
    - 🗄️ Register database health checks (DynamoDB connectivity)
    - 🔴 Register Redis health checks (all cluster connectivity)
    - 🌐 Register external service checks (third-party dependencies)

**Result:** ✅ Complete monitoring and observability infrastructure active

## 🎬 PHASE 7: Final Startup & Verification

### Step 13: Managed Objects Startup

**Parallel service startup coordination:**

#### Startup Group A:

- ♻️ **Lifecycle Manager** starts:
    - ▶️ Redis connections activation
    - ▶️ DynamoDB connections activation
    - ▶️ Executor services activation

#### Startup Group B (Simultaneous):

- ♻️ **Lifecycle Manager** starts:
    - ▶️ Business services activation
    - ▶️ HTTP/WebSocket/gRPC servers activation

### Step 14: Startup Health Verification

**Comprehensive health verification:**

- 💚 **Health Checks** performs parallel verification:
    - ✅ Verify Redis connectivity across all clusters
    - ✅ Verify DynamoDB connectivity across all tables
    - ✅ Verify all business service health status

### Step 15: Startup Completion

- **WhisperServerService** confirms: 💚 System healthy
- **Main Application** receives: 🎉 Application startup complete
- **Status:** Signal server ready to handle production traffic

## 🔄 RUNTIME: Normal Operation

### Step 16: Continuous Monitoring

**Ongoing system health monitoring:**

#### Parallel Monitoring Operations:

- **Health Monitoring:**
    - 💓 Periodic Redis health checks
    - 💓 Periodic DynamoDB health checks
    - 💓 Business service health monitoring

- **Metrics & Configuration:**
    - 📊 Continuous metrics collection and export
    - 🔄 Configuration change polling and updates
    - 👀 Managed object health supervision

## 🛑 SHUTDOWN: Graceful Termination

### Step 17: Shutdown Signal Processing

- 🚀 **Main Application** receives shutdown signal (SIGTERM, SIGINT)
- Calls **WhisperServerService** shutdown method
- **Lifecycle Manager** initiates graceful shutdown sequence

### Step 18: Graceful Service Termination

**Ordered shutdown sequence:**

1. **Request Handling Shutdown:**
    - 🚫 Stop accepting new HTTP/WebSocket requests
    - ⏳ Allow in-flight requests to complete (grace period)

2. **Service Shutdown:**
    - 🛑 Stop business services in dependency order
    - 🔌 Shutdown executor services with proper thread termination

3. **Connection Cleanup:**
    - ❌ Close Redis connections gracefully
    - ❌ Close DynamoDB connections gracefully

### Step 19: Shutdown Completion

- **Lifecycle Manager** confirms: ✅ Shutdown complete
- **Main Application** receives: 🏁 Application stopped
- **Result:** Clean application termination with no resource leaks


