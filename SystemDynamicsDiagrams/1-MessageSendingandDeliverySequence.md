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