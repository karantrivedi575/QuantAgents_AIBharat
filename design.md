# PaySetu Design Document

## Overview

PaySetu is an offline-first payment ecosystem that enables peer-to-peer financial transactions in zero-connectivity environments. The system architecture is built around three core principles:

1. **Offline-First Operation**: All critical transaction functionality works without internet connectivity
2. **Hardware-Backed Security**: Cryptographic operations leverage Android's Trusted Execution Environment (TEE)
3. **Eventual Consistency**: Transactions sync to cloud infrastructure when connectivity is restored

The system consists of:
- **Mobile Application Layer**: Native Android app with offline-first design
- **P2P Communication Layer**: Google Nearby Connections API for device discovery and data transfer
- **Local Data Layer**: Encrypted SQLite database for transaction storage
- **Security Layer**: Android Keystore System for cryptographic operations
- **Cloud Sync Layer**: AWS serverless infrastructure for reconciliation

### Key Design Decisions

**Why Google Nearby Connections API?**
- Abstracts complexity of Bluetooth/Wi-Fi Direct switching
- Automatic protocol selection based on device capabilities
- Battle-tested by Google for offline gaming and file sharing
- Handles connection lifecycle and error recovery

**Why Android Keystore System?**
- Hardware-backed key storage in TEE (when available)
- Keys never leave secure hardware
- Cryptographic operations performed in isolated environment
- Protection against root access and memory dumps

**Why SQLCipher?**
- Full database encryption at rest
- Transparent encryption/decryption
- Minimal performance overhead
- Industry-standard AES-256 encryption

**Why AWS Lambda + DynamoDB?**
- Serverless architecture scales automatically
- Pay-per-use pricing model
- DynamoDB handles high write throughput
- Global tables for multi-region support

## Architecture

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile Application                       │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────────┐   │
│  │ UI Layer   │  │ Business   │  │  Sync Manager       │   │
│  │ (Jetpack   │──│ Logic      │──│  (Background)       │   │
│  │  Compose)  │  │ (ViewModels│  │                     │   │
│  └────────────┘  └────────────┘  └─────────────────────┘   │
│         │              │                    │                │
│  ┌──────▼──────────────▼────────────────────▼──────────┐   │
│  │           Repository Layer (Data Abstraction)        │   │
│  └──────┬──────────────┬────────────────────┬──────────┘   │
│         │              │                    │                │
│  ┌──────▼──────┐ ┌─────▼─────┐      ┌──────▼──────────┐   │
│  │  P2P Comm   │ │  Local DB │      │  Keystore       │   │
│  │  Manager    │ │ (SQLCipher│      │  Manager        │   │
│  └──────┬──────┘ └─────┬─────┘      └──────┬──────────┘   │
└─────────┼──────────────┼────────────────────┼──────────────┘
          │              │                    │
┌─────────▼──────┐ ┌─────▼─────┐      ┌──────▼──────────┐
│ Google Nearby  │ │ SQLCipher │      │ Android         │
│ Connections    │ │ Database  │      │ Keystore        │
│ API            │ │ Engine    │      │ System (TEE)    │
└─────────┬──────┘ └───────────┘      └─────────────────┘
          │
┌─────────▼──────────────────────────────────────────────┐
│  Transport Layer (Wi-Fi Direct / Bluetooth)            │
└────────────────────────────────────────────────────────┘

          │ (When online)
          │
┌─────────▼──────────────────────────────────────────────┐
│              AWS Cloud Infrastructure                   │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────┐  │
│  │ API        │  │ Lambda     │  │  DynamoDB       │  │
│  │ Gateway    │──│ Functions  │──│  (Global        │  │
│  │            │  │            │  │   Ledger)       │  │
│  └────────────┘  └────────────┘  └─────────────────┘  │
│                                   ┌─────────────────┐  │
│                                   │  S3 (Encrypted  │  │
│                                   │   Backups)      │  │
│                                   └─────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### Component Responsibilities

#### UI Layer
- Transaction initiation and confirmation screens
- Device discovery visualization
- Transaction history display
- Sync status indicators
- Error messaging and user feedback

#### Business Logic Layer
- Transaction validation rules
- Amount formatting and validation
- Transaction state management
- Sync queue management
- Conflict resolution logic

#### Repository Layer
- Abstract data access patterns
- Coordinate between local DB and cloud
- Handle data transformation
- Manage caching strategies

#### P2P Communication Manager
- Device discovery and advertising
- Connection establishment and lifecycle
- Secure data transmission
- Protocol negotiation (Wi-Fi Direct vs Bluetooth)
- Connection quality monitoring

#### Local Database Manager
- Transaction CRUD operations
- Query optimization
- Database encryption key management
- Schema migrations
- Data integrity checks

#### Keystore Manager
- Key generation and storage
- Cryptographic signing operations
- Key rotation policies
- Hardware capability detection

#### Sync Manager
- Network connectivity monitoring
- Batch transaction preparation
- Retry logic with exponential backoff
- Conflict detection and resolution
- Sync status tracking

## Components and Interfaces

### Core Components

#### 1. TransactionManager

**Responsibility**: Orchestrates the complete transaction lifecycle

```kotlin
interface TransactionManager {
    /**
     * Initiates a new payment transaction
     * @param amount Transaction amount in smallest currency unit (e.g., paise)
     * @param recipientId Unique identifier of the recipient
     * @return TransactionResult with transaction ID or error
     */
    suspend fun initiatePayment(
        amount: Long,
        recipientId: String
    ): Result<TransactionId>
    
    /**
     * Accepts an incoming payment request
     * @param transactionId ID of the pending transaction
     * @return Result indicating success or failure
     */
    suspend fun acceptPayment(transactionId: TransactionId): Result<Unit>
    
    /**
     * Retrieves transaction by ID
     */
    suspend fun getTransaction(id: TransactionId): Result<Transaction>
    
    /**
     * Lists all transactions with optional filtering
     */
    suspend fun listTransactions(
        filter: TransactionFilter = TransactionFilter.All,
        limit: Int = 50
    ): Result<List<Transaction>>
}
```

#### 2. P2PConnectionManager

**Responsibility**: Manages peer-to-peer device connections

```kotlin
interface P2PConnectionManager {
    /**
     * Starts advertising this device for discovery
     * @param userProfile User information to broadcast
     * @return Result with advertising session or error
     */
    suspend fun startAdvertising(
        userProfile: UserProfile
    ): Result<AdvertisingSession>
    
    /**
     * Starts discovering nearby devices
     * @return Flow of discovered devices
     */
    fun startDiscovery(): Flow<DiscoveredDevice>
    
    /**
     * Establishes connection to a discovered device
     * @param deviceId ID of the device to connect to
     * @return Result with established connection or error
     */
    suspend fun connect(deviceId: String): Result<P2PConnection>
    
    /**
     * Sends data over an established connection
     * @param connection Active P2P connection
     * @param payload Data to send
     * @return Result indicating success or failure
     */
    suspend fun sendData(
        connection: P2PConnection,
        payload: ByteArray
    ): Result<Unit>
    
    /**
     * Receives data from connection
     * @param connection Active P2P connection
     * @return Flow of incoming data packets
     */
    fun receiveData(connection: P2PConnection): Flow<ByteArray>
    
    /**
     * Closes an active connection
     */
    suspend fun disconnect(connection: P2PConnection)
    
    /**
     * Stops all advertising and discovery
     */
    suspend fun stopAll()
}
```

#### 3. CryptoManager

**Responsibility**: Handles all cryptographic operations

```kotlin
interface CryptoManager {
    /**
     * Generates a new key pair in Android Keystore
     * @param alias Unique identifier for the key
     * @return Result indicating success or failure
     */
    suspend fun generateKeyPair(alias: String): Result<Unit>
    
    /**
     * Signs data using private key from Keystore
     * @param alias Key alias
     * @param data Data to sign
     * @return Result with signature bytes or error
     */
    suspend fun sign(alias: String, data: ByteArray): Result<ByteArray>
    
    /**
     * Verifies signature using public key
     * @param publicKey Public key for verification
     * @param data Original data
     * @param signature Signature to verify
     * @return Result indicating if signature is valid
     */
    suspend fun verify(
        publicKey: ByteArray,
        data: ByteArray,
        signature: ByteArray
    ): Result<Boolean>
    
    /**
     * Encrypts data using AES-GCM
     * @param data Data to encrypt
     * @param key Encryption key
     * @return Result with encrypted data or error
     */
    suspend fun encrypt(data: ByteArray, key: ByteArray): Result<EncryptedData>
    
    /**
     * Decrypts data using AES-GCM
     * @param encryptedData Data to decrypt
     * @param key Decryption key
     * @return Result with decrypted data or error
     */
    suspend fun decrypt(encryptedData: EncryptedData, key: ByteArray): Result<ByteArray>
    
    /**
     * Generates a secure random key
     * @param lengthBytes Key length in bytes
     * @return Cryptographically secure random bytes
     */
    fun generateSecureRandom(lengthBytes: Int): ByteArray
    
    /**
     * Checks if device has hardware-backed keystore
     * @return true if TEE is available
     */
    fun hasHardwareBackedKeystore(): Boolean
}
```

#### 4. LocalLedgerManager

**Responsibility**: Manages local encrypted transaction database

```kotlin
interface LocalLedgerManager {
    /**
     * Stores a new transaction in local ledger
     * @param transaction Transaction to store
     * @return Result with stored transaction ID or error
     */
    suspend fun storeTransaction(transaction: Transaction): Result<TransactionId>
    
    /**
     * Updates existing transaction
     * @param transaction Transaction with updated fields
     * @return Result indicating success or failure
     */
    suspend fun updateTransaction(transaction: Transaction): Result<Unit>
    
    /**
     * Retrieves transaction by ID
     */
    suspend fun getTransaction(id: TransactionId): Result<Transaction>
    
    /**
     * Queries transactions with filters
     */
    suspend fun queryTransactions(
        query: TransactionQuery
    ): Result<List<Transaction>>
    
    /**
     * Gets all unsynced transactions
     * @return List of transactions pending cloud sync
     */
    suspend fun getUnsyncedTransactions(): Result<List<Transaction>>
    
    /**
     * Marks transactions as synced
     * @param transactionIds IDs of synced transactions
     */
    suspend fun markAsSynced(transactionIds: List<TransactionId>): Result<Unit>
    
    /**
     * Calculates current balance from local ledger
     * @return Current balance in smallest currency unit
     */
    suspend fun calculateBalance(): Result<Long>
    
    /**
     * Initializes database with encryption key
     * @param encryptionKey Key for SQLCipher
     */
    suspend fun initialize(encryptionKey: ByteArray): Result<Unit>
}
```

#### 5. CloudSyncManager

**Responsibility**: Synchronizes local transactions with cloud infrastructure

```kotlin
interface CloudSyncManager {
    /**
     * Syncs all pending transactions to cloud
     * @return Result with sync summary or error
     */
    suspend fun syncToCloud(): Result<SyncSummary>
    
    /**
     * Pulls latest transactions from cloud
     * @param since Timestamp to fetch transactions after
     * @return Result with fetched transactions or error
     */
    suspend fun pullFromCloud(since: Timestamp): Result<List<Transaction>>
    
    /**
     * Resolves conflicts between local and cloud transactions
     * @param conflicts List of conflicting transactions
     * @return Result with resolved transactions
     */
    suspend fun resolveConflicts(
        conflicts: List<TransactionConflict>
    ): Result<List<Transaction>>
    
    /**
     * Monitors network connectivity
     * @return Flow emitting connectivity status changes
     */
    fun observeConnectivity(): Flow<ConnectivityStatus>
    
    /**
     * Enables automatic sync when online
     */
    suspend fun enableAutoSync()
    
    /**
     * Disables automatic sync
     */
    suspend fun disableAutoSync()
}
```

### Data Transfer Objects

#### TransactionPayload

```kotlin
/**
 * Wire format for P2P transaction transmission
 */
data class TransactionPayload(
    val version: Int = 1,
    val transactionId: String,
    val senderId: String,
    val senderPublicKey: ByteArray,
    val recipientId: String,
    val amount: Long,
    val timestamp: Long,
    val signature: ByteArray,
    val metadata: Map<String, String> = emptyMap()
) {
    fun toByteArray(): ByteArray
    companion object {
        fun fromByteArray(bytes: ByteArray): TransactionPayload
    }
}
```

#### SyncPayload

```kotlin
/**
 * Batch payload for cloud synchronization
 */
data class SyncPayload(
    val deviceId: String,
    val transactions: List<Transaction>,
    val checksum: String,
    val timestamp: Long,
    val signature: ByteArray
)
```

## Data Models

### Core Domain Models

#### Transaction

```kotlin
data class Transaction(
    val id: TransactionId,
    val senderId: UserId,
    val recipientId: UserId,
    val amount: Long, // Amount in smallest currency unit (paise)
    val timestamp: Timestamp,
    val status: TransactionStatus,
    val signature: ByteArray,
    val syncStatus: SyncStatus,
    val createdAt: Timestamp,
    val updatedAt: Timestamp,
    val metadata: TransactionMetadata
)

enum class TransactionStatus {
    PENDING,      // Initiated but not confirmed
    CONFIRMED,    // Both parties confirmed
    FAILED,       // Transaction failed
    CANCELLED     // Transaction cancelled
}

enum class SyncStatus {
    NOT_SYNCED,   // Not yet synced to cloud
    SYNCING,      // Currently syncing
    SYNCED,       // Successfully synced
    SYNC_FAILED   // Sync failed, will retry
}

data class TransactionMetadata(
    val deviceModel: String,
    val appVersion: String,
    val connectionType: String, // "wifi_direct" or "bluetooth"
    val notes: String? = null
)
```

#### User Profile

```kotlin
data class UserProfile(
    val id: UserId,
    val displayName: String,
    val phoneNumber: String,
    val publicKey: ByteArray,
    val deviceId: String,
    val createdAt: Timestamp
)
```

#### P2P Connection

```kotlin
data class P2PConnection(
    val connectionId: String,
    val remoteDeviceId: String,
    val remoteUserProfile: UserProfile,
    val connectionType: ConnectionType,
    val establishedAt: Timestamp,
    val quality: ConnectionQuality
)

enum class ConnectionType {
    WIFI_DIRECT,
    BLUETOOTH,
    BLE
}

data class ConnectionQuality(
    val signalStrength: Int, // 0-100
    val latencyMs: Long,
    val bandwidth: Long // bytes per second
)
```

#### Discovered Device

```kotlin
data class DiscoveredDevice(
    val deviceId: String,
    val userProfile: UserProfile,
    val distance: Distance,
    val discoveredAt: Timestamp
)

enum class Distance {
    IMMEDIATE,  // < 1 meter
    NEAR,       // 1-10 meters
    FAR         // > 10 meters
}
```

### Database Schema

#### Transactions Table

```sql
CREATE TABLE transactions (
    id TEXT PRIMARY KEY,
    sender_id TEXT NOT NULL,
    recipient_id TEXT NOT NULL,
    amount INTEGER NOT NULL,
    timestamp INTEGER NOT NULL,
    status TEXT NOT NULL,
    signature BLOB NOT NULL,
    sync_status TEXT NOT NULL,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    metadata_json TEXT NOT NULL,
    CONSTRAINT amount_positive CHECK (amount > 0),
    CONSTRAINT valid_status CHECK (status IN ('PENDING', 'CONFIRMED', 'FAILED', 'CANCELLED')),
    CONSTRAINT valid_sync_status CHECK (sync_status IN ('NOT_SYNCED', 'SYNCING', 'SYNCED', 'SYNC_FAILED'))
);

CREATE INDEX idx_transactions_sender ON transactions(sender_id);
CREATE INDEX idx_transactions_recipient ON transactions(recipient_id);
CREATE INDEX idx_transactions_timestamp ON transactions(timestamp DESC);
CREATE INDEX idx_transactions_sync_status ON transactions(sync_status);
CREATE INDEX idx_transactions_status ON transactions(status);
```

#### Users Table

```sql
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    display_name TEXT NOT NULL,
    phone_number TEXT NOT NULL UNIQUE,
    public_key BLOB NOT NULL,
    device_id TEXT NOT NULL,
    created_at INTEGER NOT NULL
);

CREATE INDEX idx_users_phone ON users(phone_number);
```

#### Sync Queue Table

```sql
CREATE TABLE sync_queue (
    transaction_id TEXT PRIMARY KEY,
    retry_count INTEGER NOT NULL DEFAULT 0,
    last_attempt_at INTEGER,
    next_retry_at INTEGER,
    error_message TEXT,
    FOREIGN KEY (transaction_id) REFERENCES transactions(id) ON DELETE CASCADE
);

CREATE INDEX idx_sync_queue_next_retry ON sync_queue(next_retry_at);
```

### Cloud Data Models (DynamoDB)

#### Global Ledger Table

```
Table: PaySetuTransactions
Partition Key: userId (String)
Sort Key: timestamp (Number)

Attributes:
- transactionId (String)
- senderId (String)
- recipientId (String)
- amount (Number)
- timestamp (Number)
- status (String)
- signature (Binary)
- deviceId (String)
- metadata (Map)

Global Secondary Indexes:
1. TransactionIdIndex
   - Partition Key: transactionId
   - For quick lookup by transaction ID

2. RecipientIndex
   - Partition Key: recipientId
   - Sort Key: timestamp
   - For querying received transactions
```

### Type Aliases

```kotlin
typealias TransactionId = String
typealias UserId = String
typealias Timestamp = Long // Unix timestamp in milliseconds
typealias DeviceId = String
```


### API Specifications

#### P2P Transaction Protocol

The P2P transaction protocol defines the message exchange between payer and payee devices.

**Message Flow:**

```
Payer Device                                    Payee Device
     |                                                |
     |  1. DISCOVERY_BROADCAST                       |
     |<-----------------------------------------------|
     |                                                |
     |  2. CONNECTION_REQUEST                         |
     |----------------------------------------------->|
     |                                                |
     |  3. CONNECTION_ACCEPT + PUBLIC_KEY             |
     |<-----------------------------------------------|
     |                                                |
     |  4. TRANSACTION_INIT                           |
     |     {amount, senderId, timestamp, signature}   |
     |----------------------------------------------->|
     |                                                |
     |  5. TRANSACTION_ACCEPT + SIGNATURE             |
     |<-----------------------------------------------|
     |                                                |
     |  6. TRANSACTION_COMMIT                         |
     |----------------------------------------------->|
     |                                                |
     |  7. TRANSACTION_RECEIPT                        |
     |<-----------------------------------------------|
     |                                                |
     |  8. CONNECTION_CLOSE                           |
     |<---------------------------------------------->|
```

**Message Formats:**

```kotlin
// Message 1: Discovery Broadcast
data class DiscoveryBroadcast(
    val messageType: String = "DISCOVERY",
    val userId: String,
    val displayName: String,
    val publicKey: ByteArray,
    val deviceId: String
)

// Message 4: Transaction Init
data class TransactionInit(
    val messageType: String = "TRANSACTION_INIT",
    val transactionId: String,
    val senderId: String,
    val recipientId: String,
    val amount: Long,
    val timestamp: Long,
    val signature: ByteArray
)

// Message 5: Transaction Accept
data class TransactionAccept(
    val messageType: String = "TRANSACTION_ACCEPT",
    val transactionId: String,
    val recipientSignature: ByteArray,
    val timestamp: Long
)

// Message 6: Transaction Commit
data class TransactionCommit(
    val messageType: String = "TRANSACTION_COMMIT",
    val transactionId: String,
    val finalSignature: ByteArray
)

// Message 7: Transaction Receipt
data class TransactionReceipt(
    val messageType: String = "TRANSACTION_RECEIPT",
    val transactionId: String,
    val senderId: String,
    val recipientId: String,
    val amount: Long,
    val timestamp: Long,
    val senderSignature: ByteArray,
    val recipientSignature: ByteArray
)
```

#### Cloud Sync API

**Base URL:** `https://api.paysetu.com/v1`

**Authentication:** Bearer token (JWT) in Authorization header

**Endpoints:**

##### POST /sync/transactions

Uploads batch of local transactions to cloud.

**Request:**
```json
{
  "deviceId": "device-uuid",
  "transactions": [
    {
      "id": "txn-uuid",
      "senderId": "user-uuid",
      "recipientId": "user-uuid",
      "amount": 10000,
      "timestamp": 1708012800000,
      "status": "CONFIRMED",
      "signature": "base64-encoded-signature",
      "metadata": {
        "deviceModel": "Pixel 7",
        "appVersion": "1.0.0",
        "connectionType": "wifi_direct"
      }
    }
  ],
  "checksum": "sha256-hash",
  "signature": "base64-encoded-signature"
}
```

**Response:**
```json
{
  "success": true,
  "syncedCount": 1,
  "conflicts": [],
  "timestamp": 1708012900000
}
```

##### GET /sync/transactions

Fetches transactions from cloud since a given timestamp.

**Query Parameters:**
- `since`: Unix timestamp in milliseconds
- `limit`: Maximum number of transactions (default: 100, max: 1000)

**Response:**
```json
{
  "transactions": [
    {
      "id": "txn-uuid",
      "senderId": "user-uuid",
      "recipientId": "user-uuid",
      "amount": 10000,
      "timestamp": 1708012800000,
      "status": "CONFIRMED",
      "signature": "base64-encoded-signature"
    }
  ],
  "hasMore": false,
  "nextCursor": null
}
```

##### POST /sync/resolve-conflicts

Resolves transaction conflicts between local and cloud.

**Request:**
```json
{
  "conflicts": [
    {
      "transactionId": "txn-uuid",
      "localVersion": { /* transaction object */ },
      "cloudVersion": { /* transaction object */ },
      "resolution": "KEEP_CLOUD" // or "KEEP_LOCAL" or "MERGE"
    }
  ]
}
```

**Response:**
```json
{
  "resolved": [
    {
      "transactionId": "txn-uuid",
      "finalVersion": { /* resolved transaction */ }
    }
  ]
}
```

### Transaction State Machine

```
                    ┌─────────┐
                    │ PENDING │
                    └────┬────┘
                         │
              ┌──────────┼──────────┐
              │                     │
         accept()              cancel()
              │                     │
              ▼                     ▼
        ┌───────────┐         ┌───────────┐
        │ CONFIRMED │         │ CANCELLED │
        └───────────┘         └───────────┘
              │
         timeout() or
         validation_fail()
              │
              ▼
        ┌──────────┐
        │  FAILED  │
        └──────────┘
```

**State Transitions:**

- `PENDING → CONFIRMED`: Both parties sign and accept transaction
- `PENDING → CANCELLED`: Either party cancels before confirmation
- `PENDING → FAILED`: Timeout (30 seconds) or validation failure
- `CONFIRMED → FAILED`: Post-confirmation validation fails (rare)

**Invariants:**
- Once `CONFIRMED`, transaction cannot transition to `CANCELLED`
- `FAILED` and `CANCELLED` are terminal states
- Amount must remain constant across all states
- Signatures must be valid for state transitions

### Security Architecture

#### Cryptographic Specifications

**Key Generation:**
- Algorithm: RSA-2048 or EC P-256 (based on hardware support)
- Storage: Android Keystore System (hardware-backed when available)
- Key Usage: Digital signatures only (not for encryption)
- Key Rotation: Every 90 days (configurable)

**Transaction Signing:**
- Algorithm: SHA-256 with RSA or ECDSA
- Signed Data: `transactionId || senderId || recipientId || amount || timestamp`
- Signature Format: DER-encoded

**Database Encryption:**
- Algorithm: AES-256-GCM
- Key Derivation: PBKDF2 with 100,000 iterations
- Salt: 32 bytes, randomly generated per device
- Key Storage: Android Keystore (encrypted with hardware key)

**Transport Security:**
- P2P: No additional encryption (physical proximity assumed secure)
- Cloud Sync: TLS 1.3 with certificate pinning
- API Authentication: JWT with 1-hour expiration

#### Threat Model

**Threats Addressed:**

1. **Device Theft/Loss**
   - Mitigation: Database encryption, biometric/PIN lock
   - Impact: Attacker cannot read transaction history

2. **Man-in-the-Middle (P2P)**
   - Mitigation: Cryptographic signatures, visual confirmation
   - Impact: Attacker cannot modify transaction amount

3. **Replay Attacks**
   - Mitigation: Timestamp validation, transaction ID uniqueness
   - Impact: Old transactions cannot be replayed

4. **Double Spending**
   - Mitigation: Local balance checks, cloud reconciliation
   - Impact: User cannot spend more than available balance

5. **Transaction Repudiation**
   - Mitigation: Dual signatures (sender + recipient)
   - Impact: Neither party can deny transaction occurred

**Threats Not Addressed:**

1. **Physical Coercion**: User forced to make payment
2. **Social Engineering**: User tricked into sending money
3. **Device Compromise**: Rooted device with malware
4. **Quantum Computing**: Future threat to RSA/ECC

#### Security Best Practices

- Minimum Android version: 8.0 (API 26) for security features
- Mandatory biometric or PIN authentication for transactions > ₹500
- Transaction limits: ₹10,000 per transaction, ₹50,000 per day (offline)
- Automatic logout after 5 minutes of inactivity
- Certificate pinning for cloud API
- Obfuscation with R8/ProGuard
- Root detection (warning only, not blocking)

### Performance Considerations

#### Latency Targets

- Device discovery: < 2 seconds
- Connection establishment: < 3 seconds
- Transaction completion: < 5 seconds (end-to-end)
- Database query: < 50ms (for 10,000 transactions)
- Sync operation: < 10 seconds (for 100 transactions)

#### Optimization Strategies

**P2P Communication:**
- Prefer Wi-Fi Direct over Bluetooth (10x faster)
- Use connection pooling for multiple transactions
- Implement message compression for large payloads
- Timeout aggressive connections (30 seconds)

**Database:**
- Use prepared statements for all queries
- Index frequently queried columns
- Batch inserts for sync operations
- Vacuum database weekly
- Limit query results with pagination

**Cryptography:**
- Cache public keys to avoid repeated Keystore access
- Use hardware acceleration when available
- Batch signature operations
- Lazy-load encryption keys

**Memory:**
- Limit in-memory transaction cache to 100 items
- Use Kotlin Flow for streaming large datasets
- Release connections immediately after use
- Implement LRU cache for user profiles

#### Scalability

**Local Storage:**
- Maximum transactions: 100,000 per device
- Database size limit: 500 MB
- Automatic archival of transactions > 1 year old

**Cloud Infrastructure:**
- DynamoDB auto-scaling: 1-10,000 WCU/RCU
- Lambda concurrency: 1,000 concurrent executions
- API Gateway throttling: 10,000 requests/second
- S3 backup retention: 7 years

### Offline Behavior

#### Offline Capabilities

**Fully Functional Offline:**
- Device discovery and connection
- Transaction initiation and confirmation
- Transaction history viewing
- Balance calculation (local ledger)
- Receipt generation

**Requires Connectivity:**
- Initial user registration
- Cloud sync
- Balance reconciliation with cloud
- Dispute resolution
- App updates

#### Offline Limits

- Maximum offline transactions: 1,000 (before sync required)
- Maximum offline duration: 30 days (before re-authentication)
- Transaction limit (offline): ₹10,000 per transaction
- Daily limit (offline): ₹50,000

#### Sync Strategy

**Automatic Sync Triggers:**
- Network connectivity detected
- App brought to foreground
- Every 6 hours (if online)
- After 10 new offline transactions

**Sync Priority:**
- High: Confirmed transactions
- Medium: Failed transactions (for analytics)
- Low: Cancelled transactions

**Conflict Resolution:**
- Server timestamp wins for status conflicts
- Higher amount wins for amount conflicts (rare)
- Manual resolution for signature mismatches
- Duplicate detection by transaction ID


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Offline Transaction Completion

*For any* valid transaction between two devices with P2P connectivity, the transaction should complete successfully without requiring internet connectivity.

**Validates: Requirements US-1.2, US-3.2**

**Test Strategy**: Generate random valid transactions (varying amounts, user IDs, device types) and execute them with network connectivity disabled. Verify all transactions reach CONFIRMED status.

### Property 2: Dual Ledger Consistency

*For any* completed transaction, both the sender's and recipient's local ledgers should contain identical transaction records (same ID, amount, timestamp, and signatures).

**Validates: Requirements US-1.3**

**Test Strategy**: Generate random transactions, execute them, then query both sender and recipient ledgers. Verify the transaction data matches exactly on both sides.

### Property 3: Local Transaction Persistence

*For any* transaction that reaches CONFIRMED status, querying the local database should return that transaction with all its original data intact.

**Validates: Requirements US-1.4, US-2.3**

**Test Strategy**: Generate random transactions, store them in the local ledger, then retrieve them. Verify all fields match the original transaction data (round-trip property).

### Property 4: Device Discovery

*For any* device that is advertising its presence, it should appear in the discovery results of nearby devices within the discovery timeout period.

**Validates: Requirements US-2.1**

**Test Strategy**: Generate random user profiles, start advertising, initiate discovery from another device. Verify the advertising device appears in discovery results.

### Property 5: Cryptographic Signature Validity

*For any* transaction in the system, both the sender's and recipient's signatures should be cryptographically valid when verified against their respective public keys and the transaction data.

**Validates: Requirements US-2.2**

**Test Strategy**: Generate random transactions with signatures, verify each signature using the corresponding public key. All signatures must pass verification.

### Property 6: Cloud Sync Reconciliation

*For any* set of local transactions that are synced to the cloud, querying the cloud ledger should return all those transactions with matching data.

**Validates: Requirements US-2.4**

**Test Strategy**: Generate random transactions, sync them to cloud, then fetch from cloud. Verify all transactions are present in cloud with identical data.

### Property 7: Concurrent Transaction Handling

*For any* set of concurrent transactions (multiple transactions happening simultaneously), all transactions should complete successfully without data corruption or race conditions.

**Validates: Requirements US-3.1**

**Test Strategy**: Generate multiple random transactions, execute them concurrently using multiple threads/coroutines. Verify all complete with CONFIRMED status and no data corruption in the ledger.

### Property 8: Automatic Sync Trigger

*For any* connectivity state change from offline to online, the sync manager should automatically initiate a sync operation for pending transactions.

**Validates: Requirements US-4.1**

**Test Strategy**: Generate random transactions while offline, then simulate connectivity restoration. Verify sync is automatically triggered and completes.

### Property 9: Sync Payload Encryption

*For any* sync operation, the payload transmitted to the cloud should be encrypted before transmission.

**Validates: Requirements US-4.2**

**Test Strategy**: Generate random transactions, initiate sync, intercept the payload before transmission. Verify the payload is encrypted (cannot read plaintext transaction data).

### Property 10: Conflict Resolution

*For any* pair of conflicting transactions (same transaction ID with different data), the conflict resolution algorithm should produce a deterministic result based on the defined resolution rules.

**Validates: Requirements US-4.3**

**Test Strategy**: Generate random transaction conflicts (same ID, different amounts or timestamps), apply conflict resolution. Verify the resolution follows the rules (server timestamp wins, higher amount wins, etc.).

### Property 11: Transaction Integrity Through Sync

*For any* transaction, syncing to cloud and then fetching back should produce an equivalent transaction (round-trip property).

**Validates: Requirements US-4.4**

**Test Strategy**: Generate random transactions, sync to cloud, fetch from cloud. Verify the fetched transaction matches the original (amount, parties, timestamp, signatures all identical).

### Property 12: Balance Invariant

*For any* set of transactions in the system, the sum of all debits should equal the sum of all credits across all users (zero-sum property).

**Validates: System-wide correctness**

**Test Strategy**: Generate random sequences of transactions between multiple users, calculate total debits and credits. Verify they sum to zero (money is neither created nor destroyed).

### Property 13: Transaction ID Uniqueness

*For any* two transactions in the system, they should have different transaction IDs (no duplicates).

**Validates: System-wide correctness**

**Test Strategy**: Generate large numbers of random transactions, collect all transaction IDs. Verify no duplicates exist in the set.

### Property 14: Amount Positivity Constraint

*For any* transaction in the system, the amount should be greater than zero.

**Validates: System-wide correctness**

**Test Strategy**: Generate random transactions with various amounts. Verify all amounts are positive. Attempt to create transactions with zero or negative amounts and verify they are rejected.

### Property 15: Timestamp Validity

*For any* transaction, the timestamp should be greater than or equal to the user's account creation time and less than or equal to the current system time.

**Validates: System-wide correctness**

**Test Strategy**: Generate random transactions with various timestamps. Verify all timestamps fall within the valid range. Attempt to create transactions with future or pre-creation timestamps and verify they are rejected.

### Property 16: Sync Idempotence

*For any* transaction, syncing it to the cloud multiple times should have the same effect as syncing it once (idempotence property).

**Validates: System-wide correctness**

**Test Strategy**: Generate random transactions, sync them to cloud, then sync the same transactions again multiple times. Verify the cloud ledger contains exactly one copy of each transaction.

### Property 17: State Machine Validity

*For any* transaction state transition, the transition should follow the defined state machine rules (PENDING can go to CONFIRMED/CANCELLED/FAILED, but CONFIRMED cannot go to CANCELLED).

**Validates: System-wide correctness**

**Test Strategy**: Generate random transactions, attempt various state transitions (both valid and invalid). Verify valid transitions succeed and invalid transitions are rejected.

### Property 18: Signature Non-Repudiation

*For any* transaction with valid signatures from both parties, neither party should be able to deny their participation in the transaction.

**Validates: System-wide security**

**Test Strategy**: Generate random transactions with signatures, verify that the signatures can be independently verified by third parties using the public keys. Attempt to modify transaction data and verify signatures become invalid.

### Property 19: Connection Type Fallback

*For any* device pair where Wi-Fi Direct is unavailable, the system should successfully fall back to Bluetooth and complete the transaction.

**Validates: System-wide reliability**

**Test Strategy**: Generate random transactions, disable Wi-Fi Direct capability, execute transactions. Verify they complete successfully using Bluetooth as indicated in transaction metadata.

### Property 20: Database Encryption Integrity

*For any* transaction stored in the encrypted database, the encryption should not corrupt the data (decryption yields original data).

**Validates: System-wide security**

**Test Strategy**: Generate random transactions with various data (including edge cases like maximum amounts, special characters in metadata), store in encrypted database, retrieve and decrypt. Verify all data matches original.

