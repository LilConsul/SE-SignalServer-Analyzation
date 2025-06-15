# System Dynamics Diagram

## 1. Message Sending and Delivery Sequence

```mermaid
sequenceDiagram
    participant C1 as 📱 SignalServiceMessageSender
    participant LB as ⚖️ HAProxy/ELB
    participant API as 🌐 MessageResource.java
    participant AUTH as 🔐 AccountAuthenticator
    participant MSG as 📤 MessageSender.java
    participant REDIS as 🔴 Redis Cluster
    participant DYNAMO as 🗃️ DynamoDB Messages
    participant PUSH as 🔔 PushNotificationManager
    participant C2 as 📱 SignalServiceMessageReceiver

    rect rgb(240, 248, 255)
        Note over C1, C2: 📨 HTTP/2 Message Delivery Pipeline
    end

    C1 ->>+ LB: POST /v1/messages/{uuid}<br/>Authorization: Basic {base64}
    Note right of C1: Sealed sender envelope<br/>X3DH + Double Ratchet
    LB ->>+ API: Route via consistent hashing
    API ->>+ AUTH: validateCredentials(basicAuth)
    Note right of AUTH: BCrypt + timing-safe compare
    AUTH -->>- API: AuthenticatedDevice{accountId, deviceId}
    API ->>+ MSG: sendMessage(envelope, timestamp, urgent)
    Note right of MSG: Protobuf envelope validation<br/>Max 256KB content

    par Redis presence check
        MSG ->>+ REDIS: SISMEMBER online_devices {account:device}
        Note right of REDIS: Redis SET with TTL 30s
        REDIS -->>- MSG: Boolean presence
    and DynamoDB persistence
        MSG ->>+ DYNAMO: putItem(Messages table)<br/>PK: {accountId}#{deviceId}<br/>SK: timestamp#{messageId}
        Note right of DYNAMO: GSI on timestamp for cleanup
        DYNAMO -->>- MSG: ConsumedCapacity: 1 WCU
    end

    alt 🔴 Device offline (Redis miss)
        MSG ->>+ PUSH: sendNotification(apnId|fcmToken, urgent)
        Note right of PUSH: Rate limited: 100/min/device
        PUSH ->> PUSH: APNS HTTP/2 JWT auth<br/>FCM OAuth2 + exponential backoff
        PUSH -->>- MSG: DeliveryReceipt{messageId, deliveredAt}
    else 🟢 Device online (WebSocket active)
        MSG ->> C2: WebSocket BINARY frame<br/>MessageProtos.Envelope
        Note right of C2: Persistent connection<br/>Heartbeat every 55s
        C2 -->> MSG: ACK frame{messageId}
    end

    MSG -->>- API: SendMessageResponse{needsSync: boolean}
    API -->>- LB: 200 OK + timing headers
    LB -->>- C1: Response + request latency

    rect rgb(240, 255, 240)
        Note over C2, DYNAMO: 📥 Message Queue Synchronization
    end

    C2 ->>+ API: WebSocket upgrade<br/>Sec-WebSocket-Protocol: signalservice
    Note right of C2: TLS 1.3 + certificate pinning
    API ->>+ DYNAMO: query(Messages)<br/>FilterExpression: timestamp > lastSync<br/>Limit: 100, ScanIndexForward: true
    Note right of DYNAMO: Eventually consistent read<br/>Auto-scaling on RCU
    DYNAMO -->>- API: Items[]{messageId, envelope, timestamp}
    API ->> C2: WebSocket BINARY frames<br/>Batch: max 10MB or 1000 msgs
    Note right of API: Chunked delivery<br/>Back-pressure handling
    C2 -->> API: ACK batch{lastMessageId}
    Note over API: Cleanup: DELETE acknowledged messages<br/>TTL: 30 days max retention
```

## 2. 🔐 Signal Server Registration & Authentication Flow

### 2.1 📱Phone Number Verification

```mermaid
sequenceDiagram
    %% Enhanced participant styling with icons and clear roles
    participant C as 📱 Client App
    participant LB as ⚖️ Load Balancer
    participant API as 🌐 HTTP API Gateway
    participant VC as ✅ VerificationController
    participant RSC as 🎯 RegistrationServiceClient
    participant CACHE as 🔴 Redis Cache
    participant SMS as 📞 SMS/Voice Provider

    %% Phase 1: Initial Session Creation
    rect rgba(173, 216, 230, 0.15)
        Note over C,CACHE: 🚀 PHASE 1: Verification Session Setup
        
        C->>+LB: 📤 POST /v1/verification/session<br/>📋 {phoneNumber, pushToken, captcha}
        Note right of C: Rate Limited:<br/>5 requests/hour per IP
        
        LB->>+API: 🔄 Route to verification endpoint
        API->>+VC: 🎯 createSession(CreateVerificationSessionRequest)
        
        rect rgba(255, 255, 224, 0.3)
            Note over VC: Input Validation
            VC->>VC: 📋 Validate phone number format<br/>🌍 Check country code allowlist<br/>🚫 Validate against blocklist
        end
        
        VC->>+RSC: 🔗 createRegistrationSession(phoneNumber, sourceHost)
        Note right of RSC: 🔌 gRPC call to Registration Service<br/>⏱️ Timeout: 15s<br/>🔄 Retry: 3 attempts
        RSC-->>-VC: ✅ RegistrationServiceSession<br/>🆔 {sessionId, expiration: 10min}
        
        VC->>+CACHE: 💾 Store session metadata<br/>🔑 Key: session:{sessionId}<br/>⏰ TTL: 10 minutes
        CACHE-->>-VC: ✅ Session cached successfully
        
        VC-->>-API: 📤 VerificationSessionResponse<br/>🆔 {sessionId, requestedInfo[CAPTCHA]}
        API-->>-LB: ✅ HTTP 200 OK
        LB-->>-C: 🎉 Session created - Ready for verification
    end

    %% Phase 2: Code Request and Delivery
    rect rgba(255, 228, 225, 0.15)
        Note over C,CACHE: 📞 PHASE 2: Verification Code Delivery
        
        C->>+LB: 📤 PATCH /v1/verification/session/{sessionId}<br/>📋 {transport: "sms", captchaResponse}
        Note right of C: Transport options:<br/>📱 SMS, ☎️ Voice call
        
        LB->>+API: 🔄 Route code request
        API->>+VC: 🔄 updateSession(sessionId, transport, captcha)
        
        rect rgba(255, 240, 245, 0.3)
            Note over VC,CACHE: Session Validation
            VC->>+CACHE: 🔍 Validate session exists & not expired
            CACHE-->>-VC: ✅ Valid session found
            VC->>VC: 🔐 Verify CAPTCHA response<br/>🚦 Check rate limits (1 req/min per number)
        end
        
        VC->>+RSC: 📞 sendVerificationCode(sessionId, SMS, clientType)
        RSC->>+SMS: 📤 Forward to Twilio/Voice provider
        Note right of SMS: 📱 SMS: "Your Signal code: 123456"<br/>⏱️ Valid for: 10 minutes<br/>🔄 Max attempts: 3
        SMS-->>-RSC: ✅ Message queued for delivery
        RSC-->>-VC: 📋 RegistrationServiceSession(verified: false, codesSent: 1)
        
        VC->>+CACHE: 🔄 Update session state<br/>📊 {codesSent++, lastCodeSent: timestamp}
        CACHE-->>-VC: ✅ Session state updated
        
        VC-->>-API: 📤 VerificationSessionResponse<br/>📱 {transport: "sms", nextCodeIn: 60s}
        API-->>-LB: ✅ HTTP 200 OK
        LB-->>-C: 📞 Verification code sent via SMS
    end

    %% Phase 3: Code Verification
    rect rgba(240, 255, 240, 0.15)
        Note over C,CACHE: ✅ PHASE 3: Code Verification
        
        C->>+LB: 📤 PUT /v1/verification/session/{sessionId}/code<br/>📋 {verificationCode: "123456"}
        
        LB->>+API: 🔄 Route verification
        API->>+VC: ✅ verifyCode(sessionId, code)
        
        VC->>+CACHE: 🔍 Retrieve session data
        CACHE-->>-VC: 📋 Session metadata
        
        VC->>+RSC: 🔐 checkVerificationCode(sessionId, code)
        Note right of RSC: 🔍 Validate against sent code<br/>⏱️ Check expiration<br/>🔢 Max 3 attempts
        
        alt ✅ Code Valid
            RSC-->>VC: ✅ RegistrationServiceSession(verified: true)
            VC->>+CACHE: 🔄 Mark session as verified<br/>🏆 {verified: true, verifiedAt: timestamp}
            CACHE-->>-VC: ✅ Updated
            VC-->>API: 🎉 VerificationComplete{sessionId, phoneNumber}
        else ❌ Code Invalid
            RSC-->>VC: ❌ InvalidCodeException(attemptsRemaining: 2)
            VC-->>API: ⚠️ HTTP 422 Invalid Code
        else 🚫 Too Many Attempts
            RSC-->>VC: 🚫 RateLimitedException
            VC->>CACHE: 🔒 Blacklist session for 5 minutes
            VC-->>API: 🚫 HTTP 429 Too Many Requests
        end
        
        API-->>-LB: Response based on verification result
        LB-->>-C: Verification result
    end
```

### 2.2 👤 Account Registration

```mermaid
sequenceDiagram
    participant C as 📱 Client App
    participant LB as ⚖️ Load Balancer
    participant API as 🌐 HTTP API Gateway
    participant RC as 📝 RegistrationController
    participant PVTM as 🎫 PhoneVerificationTokenManager
    participant RLVM as 🔒 RegistrationLockVerificationManager
    participant AM as 👥 AccountsManager
    participant KM as 🔑 KeysManager
    participant DB as 📊 DynamoDB
    participant CACHE as 🔴 Redis Cache

    rect rgba(135, 206, 250, 0.15)
        Note over C,CACHE: 🚀 PHASE 1: Registration Request & Validation
        
        C->>+LB: 📤 POST /v1/registration<br/>📋 {sessionId, accountAttributes, identityKeys}
        Note right of C: 📦 Payload includes:<br/>🆔 Verified sessionId<br/>👤 Account attributes<br/>🔐 Identity keypairs (ACI/PNI)<br/>📱 Device capabilities
        
        LB->>+API: 🔄 Route registration request
        API->>+RC: 📝 register(RegistrationRequest)
        
        rect rgba(255, 255, 224, 0.3)
            Note over RC,PVTM: Phone Verification Check
            RC->>+PVTM: 🎫 verifyPhoneVerificationToken(sessionId, phoneNumber)
            PVTM->>PVTM: 🔍 Validate session exists<br/>✅ Check verification status<br/>⏱️ Verify not expired (16s timeout)
            PVTM-->>-RC: ✅ Phone number verified successfully
        end
    end

    rect rgba(255, 228, 225, 0.15)
        Note over C,CACHE: 🔍 PHASE 2: Existing Account Check & Registration Lock
        
        RC->>+AM: 🔍 getByE164(phoneNumber)
        AM->>+DB: 📊 Query accounts table<br/>🔑 GSI: phoneNumberIndex<br/>📱 Key: {phoneNumber}
        DB-->>-AM: 📋 Optional<Account> result
        AM-->>-RC: 👤 existingAccount (if found)
        
        alt 🔒 Existing Account with Registration Lock
            rect rgba(255, 240, 245, 0.3)
                Note over RC,RLVM: Registration Lock Security Flow
                RC->>+RLVM: 🔐 verifyRegistrationLock(account, clientPin, userAgent)
                RLVM->>RLVM: 🚦 Check rate limit: 1 attempt/5min per account<br/>📊 Get attempt history from cache
                
                alt 🔢 PIN Provided
                    RLVM->>RLVM: 🔐 Validate PIN against stored Argon2 hash<br/>🧂 Use account-specific salt
                    
                    alt ❌ PIN Invalid
                        RLVM->>RLVM: 📊 Record failed attempt<br/>📅 Calculate idle days since last seen<br/>🔄 Increment attempt counter
                        RLVM->>+CACHE: 💾 Store failed attempt<br/>🔑 Key: reglock:{uuid}<br/>⏰ TTL: 5 minutes
                        CACHE-->>-RLVM: ✅ Attempt recorded
                        RLVM-->>RC: 🚫 RateLimitExceededException(423)<br/>⏱️ {timeRemaining, attemptsLeft}
                        RC-->>API: ⚠️ HTTP 423 Registration Locked
                        API-->>LB: 📋 {timeRemaining, registrationLock: true, idleDays}
                        LB-->>C: 🔒 Account locked - Correct PIN required
                    else ✅ PIN Valid
                        RLVM->>+CACHE: 🗑️ Clear failed attempts<br/>🔑 Key: reglock:{uuid}
                        CACHE-->>-RLVM: ✅ Attempts cleared
                        RLVM-->>RC: ✅ Registration lock verified
                    end
                else 📅 7-Day Bypass Available
                    RLVM->>RLVM: 📊 Check if account idle > 7 days<br/>📅 Compare lastSeen vs current time
                    RLVM-->>RC: ✅ Bypass allowed - account dormant
                end
                RLVM-->>-RC: 🔓 Registration lock verification complete
            end
        end
    end

    rect rgba(240, 255, 240, 0.15)
        Note over C,CACHE: 👤 PHASE 3: Account Creation & Key Management
        
        RC->>+AM: 👤 create(phoneNumber, attributes, badges[], aciKey, pniKey)
        
        rect rgba(245, 255, 250, 0.3)
            Note over AM: Account Identity Generation
            AM->>AM: 🆔 Generate ACI UUID (Account Identity)<br/>📱 Generate PNI UUID (Phone Number Identity)<br/>🔐 Create Device(id: 1, authToken, salt)<br/>📊 Initialize account metrics
        end
        
        par Account Storage
            AM->>+DB: 💾 PutItem accounts table<br/>🔑 Partition Key: ACI<br/>📋 Attributes: {phoneNumber, pni, devices[], createdAt}
            Note right of DB: 📊 Account record with:<br/>🆔 ACI (primary identifier)<br/>📱 PNI (phone-linked ID)<br/>📋 Device list with capabilities<br/>🏆 Badge assignments
            DB-->>-AM: ✅ Account stored successfully
        and Key Storage
            AM->>+KM: 🔑 storeIdentityKeys(aci, pni, identityKeys)
            KM->>+DB: 💾 PutItem keys table<br/>🔑 Keys: ACI+PNI identity keypairs<br/>📝 Metadata: {algorithm, created, deviceId}
            DB-->>-KM: ✅ Identity keys stored
            KM-->>-AM: ✅ Keys management complete
        and Cache Update
            AM->>+CACHE: 💾 SET account:{ACI}<br/>📋 Full account object<br/>⏰ TTL: 3600s (1 hour)
            CACHE-->>-AM: ✅ Account cached for fast access
        end
        
        AM-->>-RC: 🎉 Account created successfully<br/>📋 {uuid: ACI, pni: PNI, deviceId: 1}
    end

    rect rgba(255, 248, 220, 0.15)
        Note over C,CACHE: 🎉 PHASE 4: Registration Completion
        
        RC-->>-API: 📤 RegistrationResponse<br/>📋 {uuid, pni, deviceId, storageCapable: true}
        Note right of API: 📦 Response includes:<br/>🆔 Account UUID (ACI)<br/>📱 Phone Number Identity (PNI)<br/>📱 Primary device ID<br/>💾 Storage capability flag
        
        API-->>-LB: ✅ HTTP 201 Created<br/>🏆 Registration successful
        LB-->>-C: 🎉 Account registration complete!<br/>🔐 Ready for secure messaging
    end
```

### 2.3 🔐 Device Authentication & Session Management

```mermaid
sequenceDiagram
    participant C as 📱 Client Device
    participant LB as ⚖️ Load Balancer
    participant API as 🌐 HTTP API Gateway
    participant AF as 🔐 AuthenticationFilter
    participant AM as 👥 AccountsManager
    participant DM as 📱 DeviceManager
    participant SM as 🎫 SessionManager
    participant CACHE as 🔴 Redis Cache
    participant DB as 📊 DynamoDB

    rect rgba(173, 216, 230, 0.15)
        Note over C,DB: 🔐 PHASE 1: Device Authentication Setup
        
        C->>+LB: 📤 PUT /v1/accounts/attributes<br/>🔐 Authorization: Basic {number}:{password}
        Note right of C: 📋 First-time device setup:<br/>📱 Device capabilities<br/>🔔 Push token registration<br/>⚙️ Client configuration
        
        LB->>+API: 🔄 Route authenticated request
        API->>+AF: 🔐 authenticateRequest(authHeader, endpoint)
        
        rect rgba(255, 255, 224, 0.3)
            Note over AF: Authentication Validation
            AF->>AF: 📋 Parse Basic Auth header<br/>📱 Extract {phoneNumber, authToken}<br/>🔍 Validate format & encoding
        end
        
        AF->>+AM: 🔍 getByE164AndAuthToken(phoneNumber, authToken)
        
        par Cache Lookup
            AM->>+CACHE: 🔍 GET account:{phoneNumber}:auth<br/>⏰ TTL check (3600s)
            CACHE-->>-AM: 📋 Cached account or cache miss
        and Database Fallback
            alt 💾 Cache Miss
                AM->>+DB: 📊 Query accounts table<br/>🔑 GSI: phoneNumberIndex<br/>🔐 Filter: authToken match
                DB-->>-AM: 📋 Account record with devices[]
                AM->>+CACHE: 💾 SET account cache<br/>⏰ TTL: 1 hour
                CACHE-->>-AM: ✅ Account cached
            end
        end
        
        AM-->>-AF: 👤 AuthenticatedAccount{uuid, deviceId, capabilities}
        AF-->>-API: ✅ Authentication successful<br/>👤 Principal: {account, device}
    end

    rect rgba(240, 255, 240, 0.15)
        Note over C,DB: 📱 PHASE 2: Device Registration & Session Creation
        
        API->>+DM: 📱 updateDeviceAttributes(account, deviceId, attributes)
        
        rect rgba(245, 255, 250, 0.3)
            Note over DM: Device State Management
            DM->>DM: 🔄 Update device capabilities<br/>📱 {GCM, APNS, WebSocket support}<br/>🔔 Update push tokens<br/>⏰ Set lastSeen timestamp
        end
        
        DM->>+SM: 🎫 createOrUpdateSession(account, device, clientInfo)
        
        rect rgba(248, 248, 255, 0.3)
            Note over SM: Session Lifecycle
            SM->>SM: 🆔 Generate session ID<br/>⏰ Set expiration (30 days)<br/>🔐 Create session token<br/>📊 Initialize session metrics
        end
        
        par Session Storage
            SM->>+CACHE: 💾 SET session:{sessionId}<br/>📋 {accountId, deviceId, created, expires}<br/>⏰ TTL: 30 days
            CACHE-->>-SM: ✅ Session cached
        and Device Update
            DM->>+DB: 🔄 UpdateItem accounts table<br/>🔑 Key: {ACI}<br/>📋 Update: devices[deviceId].{lastSeen, pushTokens}
            DB-->>-DM: ✅ Device attributes updated
        end
        
        SM-->>-DM: 🎫 SessionInfo{sessionId, token, expires}
        DM-->>-API: 📱 DeviceUpdateResponse{capabilities, sessionInfo}
    end

    rect rgba(255, 248, 220, 0.15)
        Note over C,DB: 🔄 PHASE 3: Ongoing Authentication & Session Validation
        
        loop 🔄 Subsequent API Requests
            C->>+LB: 📤 Any API Request<br/>🔐 Authorization: Bearer {sessionToken}
            LB->>+API: 🔄 Route request
            API->>+AF: 🔐 validateSession(bearerToken)
            
            AF->>+CACHE: 🔍 GET session:{tokenHash}<br/>⏰ Check expiration
            
            alt ✅ Valid Session
                CACHE-->>AF: 📋 Session data{accountId, deviceId, scope[]}
                AF->>+AM: 🔍 getAccount(accountId)
                AM->>+CACHE: 🔍 Account lookup (cached)
                CACHE-->>-AM: 👤 Account details
                AM-->>-AF: ✅ Valid account
                AF-->>API: ✅ Authenticated request<br/>👤 Principal: {account, device, session}
                
                par Session Refresh
                    AF->>+CACHE: 🔄 EXPIRE session:{tokenHash}<br/>⏰ Extend TTL by usage
                    CACHE-->>-AF: ✅ TTL extended
                and Activity Tracking
                    AF->>+CACHE: 📊 INCR activity:{accountId}:{date}<br/>⏰ TTL: 24 hours
                    CACHE-->>-AF: 📈 Activity recorded
                end
                
            else ❌ Invalid/Expired Session
                CACHE-->>AF: 🚫 Session not found or expired
                AF-->>API: ⚠️ HTTP 401 Unauthorized<br/>🔐 WWW-Authenticate: Bearer
                API-->>LB: 🚫 Authentication required
                LB-->>C: 🔐 Re-authentication needed
            end
            
            API->>API: 🎯 Process authenticated request
            API-->>-LB: 📤 API response
            LB-->>-C: ✅ Request completed
        end
    end

    rect rgba(255, 235, 235, 0.15)
        Note over C,DB: 🔄 PHASE 4: Session Cleanup & Security
        
        Note over SM,CACHE: 🧹 Background Session Management
        
        loop ⏰ Every 6 hours
            SM->>+CACHE: 🔍 SCAN expired sessions<br/>🗑️ TTL < current time
            CACHE-->>-SM: 📋 List of expired sessions
            
            SM->>+DB: 🧹 Cleanup session references<br/>🗑️ Remove from device records
            DB-->>-SM: ✅ Cleanup complete
            
            SM->>+CACHE: 📊 Collect session metrics<br/>📈 Active sessions, device types
            CACHE-->>-SM: 📊 Metrics data
        end
        
        alt 🚨 Security Event Detection
            SM->>SM: 🔍 Detect suspicious activity<br/>📱 Multiple devices, geo-anomalies
            SM->>+AM: 🚨 triggerSecurityReview(accountId, event)
            AM->>AM: 🔐 Revoke suspicious sessions<br/>📧 Send security notification
            AM-->>-SM: ✅ Security review complete
        end
    end
```
## 3. System Infrastructure and Service Management

```mermaid
sequenceDiagram
%% Enhanced styling for better visual hierarchy
    participant MAIN as 🚀 Main Application
    participant WSS as 🌐 WhisperServerService
    participant ENV as 🔧 Environment Manager
    participant LIFECYCLE as ♻️ Lifecycle Manager
    participant CONFIG as ⚙️ Configuration Manager
    participant DYNCONFIG as 🔄 DynamicConfiguration
    participant REDIS as 🔴 Redis Clusters
    participant DYNAMO as 📊 DynamoDB
    participant EXEC as ⚡ Executor Services
    participant SERVICES as 🎯 Business Services
    participant CONTROLLERS as 🎮 Controllers
    participant SERVERS as 🖥️ Server Infrastructure
    participant HEALTH as 💚 Health Checks
    participant METRICS as 📈 Metrics System
%% Color-coded sections for better visual flow
    rect rgba(135, 206, 250, 0.1)
        Note over MAIN, METRICS: 🚀 PHASE 1: System Initialization
        MAIN ->>+ WSS: 🎬 run(configuration, environment)

        rect rgba(255, 255, 224, 0.3)
            Note over WSS, CONFIG: Configuration Loading
            WSS ->>+ CONFIG: 📂 Load configuration files
            CONFIG ->> CONFIG: 📖 Parse YAML configuration
            CONFIG ->> CONFIG: ✅ Validate configuration parameters
            CONFIG -->>- WSS: ✅ Configuration loaded
        end

        rect rgba(240, 248, 255, 0.3)
            Note over WSS, LIFECYCLE: Environment Setup
            WSS ->>+ ENV: 🏗️ Initialize environment
            ENV ->>+ LIFECYCLE: 🔄 Create lifecycle manager
            LIFECYCLE -->>- ENV: ✅ Lifecycle manager ready
            ENV -->>- WSS: ✅ Environment initialized
        end
    end

    rect rgba(255, 228, 225, 0.1)
        Note over WSS, METRICS: 🔗 PHASE 2: External Dependencies Setup

        par Redis Cluster Connections
            WSS ->>+ REDIS: 🔌 Initialize Redis clusters
            rect rgba(255, 240, 245, 0.3)
                REDIS ->> REDIS: 💬 Connect to messages cluster
                REDIS ->> REDIS: 🔔 Connect to push scheduler cluster
                REDIS ->> REDIS: 🚦 Connect to rate limiters cluster
                REDIS ->> REDIS: 📡 Connect to pubsub cluster
            end
            REDIS -->>- WSS: ✅ Redis clusters connected
        and DynamoDB Connections
            WSS ->>+ DYNAMO: 🔌 Initialize DynamoDB connections
            rect rgba(248, 248, 255, 0.3)
                DYNAMO ->> DYNAMO: 👥 Configure account tables
                DYNAMO ->> DYNAMO: 🔑 Configure keys tables
                DYNAMO ->> DYNAMO: 💬 Configure messages tables
            end
            DYNAMO -->>- WSS: ✅ DynamoDB connections established
        end
    end

    rect rgba(240, 255, 240, 0.1)
        Note over WSS, METRICS: ⚡ PHASE 3: Thread Pool & Executor Setup
        WSS ->>+ EXEC: 🏭 Create executor services
        rect rgba(245, 255, 250, 0.3)
            EXEC ->> EXEC: 🌐 Create main thread pool (HTTP)
            EXEC ->> EXEC: 🔄 Create async thread pool (background)
            EXEC ->> EXEC: ⏰ Create scheduled executor (periodic tasks)
            EXEC ->> EXEC: 🔌 Create WebSocket executor
            EXEC ->> EXEC: 📬 Create message delivery executor
            EXEC ->> EXEC: 🔔 Create push notification executor
        end
        EXEC -->>- WSS: ✅ Executor services ready
    end

    rect rgba(255, 248, 220, 0.1)
        Note over WSS, METRICS: 🎯 PHASE 4: Business Services Initialization

        par Core Services
            WSS ->>+ SERVICES: 🚀 Initialize core services
            rect rgba(255, 253, 240, 0.3)
                SERVICES ->> SERVICES: 👥 Create AccountsManager
                SERVICES ->> SERVICES: 💬 Create MessagesManager
                SERVICES ->> SERVICES: 🔑 Create KeysManager
                SERVICES ->> SERVICES: 🔔 Create PushNotificationManager
                SERVICES ->> SERVICES: 📤 Create MessageSender
                SERVICES ->> SERVICES: 🔐 Create AuthenticationServices
            end
            SERVICES -->>- WSS: ✅ Core services initialized
        and Dynamic Configuration
            WSS ->>+ DYNCONFIG: 🔄 Initialize dynamic configuration
            rect rgba(250, 250, 255, 0.3)
                DYNCONFIG ->> DYNCONFIG: 🔌 Connect to configuration backend
                DYNCONFIG ->> DYNCONFIG: 🏁 Load feature flags
                DYNCONFIG ->> DYNCONFIG: 🔄 Setup configuration polling
            end
            DYNCONFIG -->>- WSS: ✅ Dynamic configuration active
        end
    end

    rect rgba(230, 230, 250, 0.1)
        Note over WSS, METRICS: 🎮 PHASE 5: API Controllers & Server Setup
        WSS ->>+ CONTROLLERS: 📋 Register REST controllers
        rect rgba(240, 240, 255, 0.3)
            CONTROLLERS ->> CONTROLLERS: 💬 Register MessageController
            CONTROLLERS ->> CONTROLLERS: 👥 Register AccountController
            CONTROLLERS ->> CONTROLLERS: 🔑 Register KeysController
            CONTROLLERS ->> CONTROLLERS: 👤 Register ProfileController
            CONTROLLERS ->> CONTROLLERS: 📱 Register DeviceController
        end
        CONTROLLERS -->>- WSS: ✅ Controllers registered
        WSS ->>+ SERVERS: 🖥️ Start server infrastructure
        rect rgba(248, 248, 255, 0.3)
            SERVERS ->> SERVERS: 🌐 Configure HTTP server (Jetty)
            SERVERS ->> SERVERS: 🔌 Configure WebSocket server
            SERVERS ->> SERVERS: ⚡ Configure gRPC servers
            SERVERS ->> SERVERS: 🔐 Setup authentication filters
            SERVERS ->> SERVERS: 🔍 Configure request filters
        end
        SERVERS -->>- WSS: ✅ Servers configured
    end

    rect rgba(240, 255, 255, 0.1)
        Note over WSS, METRICS: 📊 PHASE 6: Monitoring & Health Checks

        par Metrics System
            WSS ->>+ METRICS: 📈 Initialize metrics collection
            rect rgba(245, 255, 255, 0.3)
                METRICS ->> METRICS: 📊 Setup Micrometer registry
                METRICS ->> METRICS: 📤 Configure metric exporters
                METRICS ->> METRICS: 📝 Register custom metrics
            end
            METRICS -->>- WSS: ✅ Metrics system active
        and Health Monitoring
            WSS ->>+ HEALTH: 💚 Setup health checks
            rect rgba(240, 255, 240, 0.3)
                HEALTH ->> HEALTH: 🗄️ Register database health checks
                HEALTH ->> HEALTH: 🔴 Register Redis health checks
                HEALTH ->> HEALTH: 🌐 Register external service checks
            end
            HEALTH -->>- WSS: ✅ Health monitoring active
        end
    end

    rect rgba(255, 240, 245, 0.1)
        Note over WSS, METRICS: 🎬 PHASE 7: Final Startup & Verification
        WSS ->>+ LIFECYCLE: 🚀 Start managed objects
        rect rgba(255, 245, 250, 0.3)
            par Parallel Service Startup
                LIFECYCLE ->> REDIS: ▶️ Start Redis connections
                LIFECYCLE ->> DYNAMO: ▶️ Start DynamoDB connections
                LIFECYCLE ->> EXEC: ▶️ Start executor services
            and
                LIFECYCLE ->> SERVICES: ▶️ Start business services
                LIFECYCLE ->> SERVERS: ▶️ Start HTTP/WebSocket/gRPC servers
            end
        end
        LIFECYCLE -->>- WSS: ✅ All services started
        WSS ->>+ HEALTH: 🏥 Perform startup health check
        rect rgba(240, 255, 240, 0.3)
            par Health Verification
                HEALTH ->> REDIS: ✅ Verify Redis connectivity
                HEALTH ->> DYNAMO: ✅ Verify DynamoDB connectivity
                HEALTH ->> SERVICES: ✅ Verify service health
            end
        end
        HEALTH -->>- WSS: 💚 System healthy
        WSS -->>- MAIN: 🎉 Application startup complete
    end

    rect rgba(250, 250, 250, 0.1)
        Note over MAIN, METRICS: 🔄 RUNTIME: Normal Operation

        loop 🔄 Continuous Monitoring
            rect rgba(255, 255, 255, 0.3)
                par Ongoing Health & Metrics
                    HEALTH ->> REDIS: 💓 Periodic health checks
                    HEALTH ->> DYNAMO: 💓 Periodic health checks
                    HEALTH ->> SERVICES: 💓 Monitor service health
                and
                    METRICS ->> METRICS: 📊 Collect and export metrics
                    DYNCONFIG ->> DYNCONFIG: 🔄 Poll for configuration changes
                    LIFECYCLE ->> LIFECYCLE: 👀 Monitor managed object health
                end
            end
        end
    end

    rect rgba(255, 235, 235, 0.1)
        Note over MAIN, METRICS: 🛑 SHUTDOWN: Graceful Termination
        MAIN ->>+ WSS: 🛑 Shutdown signal received
        WSS ->>+ LIFECYCLE: 🔄 Initiate graceful shutdown

        rect rgba(255, 240, 240, 0.3)
            Note over LIFECYCLE, SERVERS: Graceful Service Termination
            LIFECYCLE ->> SERVERS: 🚫 Stop accepting new requests
            LIFECYCLE ->> SERVERS: ⏳ Complete in-flight requests
            LIFECYCLE ->> SERVICES: 🛑 Stop business services
            LIFECYCLE ->> EXEC: 🔌 Shutdown executor services
            LIFECYCLE ->> REDIS: ❌ Close Redis connections
            LIFECYCLE ->> DYNAMO: ❌ Close DynamoDB connections
        end

        LIFECYCLE -->>- WSS: ✅ Shutdown complete
        WSS -->>- MAIN: 🏁 Application stopped
    end
```