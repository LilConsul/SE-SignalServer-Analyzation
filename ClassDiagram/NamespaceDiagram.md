### This is a class diagram for the Signal Server, but instead of classes it uses namespaces to represent the structure and relationships of the system. It is 'core' of the application, showing how different components interact and are organized within the system.


```mermaid
---
config:
  class:
    hideEmptyMembersBox: true
---
classDiagram
  direction TB

  %% Core Application Layer
  class WhisperServerService {
    +initialize(bootstrap)
    +run(config, environment)
    +main(args)
    -setupControllers()
    -registerHealthChecks()
    -configureMetrics()
  }

  %% Primary Namespaces
  class textsecuregcm {
    Main application namespace
  }

  class signal {
    Extension namespace
  }

  %% Service Layers
  class API_Layer {
    REST API endpoints
    gRPC interfaces
    WebSocket endpoints
  }

  class Business_Layer {
    Business logic
    Service operations
    Features implementation
  }

  class Infrastructure_Layer {
    Data persistence
    Caching
    External integrations
  }

  %% Functional Components - API Layer
  class Controllers {
    MessageController
    ProfileController
    KeysController
    DeviceController
    AccountController
    AttachmentController
  }

  class GRPC {
    RegistrationService
    KeyTransparencyService
    SecureValueRecoveryService
  }

  class WebSocket {
    WebSocketConnectionManager
    ProvisioningManager
    MessageDeliveryManager
  }

  %% Functional Components - Business Layer
  class Managers {
    AccountsManager
    MessagesManager
    ProfilesManager
    KeysManager
    DevicesManager
  }

  class Authentication {
    AccountAuthenticator
    TurnTokenGenerator
    CredentialGenerator
  }

  class Push {
    PushNotificationManager
    APNSender
    GCMSender
    MessageSender
  }

  class Security {
    SecureStorageClient
    SecureValueRecovery
    KeyTransparency
  }

  class Experimental {
    FeatureFlags
    ABTesting
  }

  %% Functional Components - Infrastructure Layer
  class Storage {
    Accounts
    Messages
    Keys
    Profiles
    Devices
  }

  class Redis {
    ClientManager
    FaultTolerantCluster
    StateManagement
  }

  class Worker {
    DirectoryReconciliation
    MessagePersister
    CommandQueue
  }

  %% External Services
  class AWS {
    S3 (attachments)
    DynamoDB (data storage)
    CloudWatch (monitoring)
  }

  class GCP {
    FirebaseMessaging
    CloudStorage
  }

  class ExternalServices {
    Twilio (SMS)
    Stripe (payments)
    Braintree (payments)
  }

  %% Core Relationships
  WhisperServerService --> textsecuregcm : bootstraps
  WhisperServerService --> signal : initializes extensions
  textsecuregcm --> signal : provides core functionality to

  %% Layered Architecture
  textsecuregcm --> API_Layer : exposes
  textsecuregcm --> Business_Layer : implements
  textsecuregcm --> Infrastructure_Layer : utilizes

  %% API Layer Composition
  API_Layer *-- Controllers : contains
  API_Layer *-- GRPC : contains
  API_Layer *-- WebSocket : contains

  %% Business Layer Composition
  Business_Layer *-- Managers : contains
  Business_Layer *-- Authentication : contains
  Business_Layer *-- Push : contains
  Business_Layer *-- Security : contains
  Business_Layer *-- Experimental : contains

  %% Infrastructure Layer Composition
  Infrastructure_Layer *-- Storage : contains
  Infrastructure_Layer *-- Redis : contains
  Infrastructure_Layer *-- Worker : contains

  %% Key Data Flows
  Controllers --> Managers : delegates business logic to
  Managers --> Storage : persists data through
  Managers --> Redis : caches data in
  Push --> ExternalServices : delivers notifications via
  WebSocket --> Redis : manages connection state in

  %% External Integrations
  Infrastructure_Layer --> AWS : stores data in
  Infrastructure_Layer --> GCP : integrates with
  Business_Layer --> ExternalServices : uses services from

```
