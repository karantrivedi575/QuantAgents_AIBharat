---
title: PaySetu - Offline Payment Ecosystem
track: AI for Communities, Access, and Public Impact
version: 1.0
created: 2026-02-15
status: draft
---

# PaySetu: Bridging the Digital Financial Divide

## Executive Summary

PaySetu is an offline payment ecosystem designed to ensure financial access in zero-connectivity environments ("shadow zones"). By utilizing peer-to-peer communication protocols, it enables instant transaction settlement without cellular data, Wi-Fi, or OTPs, acting as a "digital vault" that remains functional when traditional payment systems fail.

## Problem Statement

### Target Challenge
Addressing the digital financial divide in shadow zones where lack of internet connectivity prevents marginalized communities and small vendors from participating in the digital economy.

### Shadow Zones Include
- Basements and underground spaces
- Rural and remote areas
- Crowded events with network congestion
- Areas with poor cellular infrastructure

### Current Pain Points
- Traditional UPI/Net Banking requires server handshake for authorization
- "Transaction Failed" errors in network dead zones
- SMS-based banking is slow, costly, and unreliable in high-traffic scenarios
- Commerce stops when connectivity fails

## Solution Overview

PaySetu removes dependency on central servers by creating direct, hardware-secured links between payer and receiver. Unlike existing solutions, it operates entirely offline with instant P2P settlement and syncs to the cloud when connectivity is restored.

## User Stories

### US-1: Basement Canteen Transaction
**As a** student in a basement canteen with no signal  
**I want to** pay for my lunch using my phone  
**So that** I don't need to carry cash or wait for network connectivity

**Acceptance Criteria:**
- Transaction completes in under 5 seconds
- No internet connection required
- Both parties receive instant confirmation
- Transaction is logged locally and synced later

### US-2: Rural Vendor Payment
**As a** small vendor at a remote community fair  
**I want to** accept digital payments without internet  
**So that** I can participate in the digital economy despite poor connectivity

**Acceptance Criteria:**
- Vendor can discover nearby payers automatically
- Payment is cryptographically secure
- Transaction history is maintained offline
- Funds are reconciled when network is available

### US-3: Crowded Event Commerce
**As a** food stall operator at a crowded festival  
**I want to** process multiple payments quickly without network congestion issues  
**So that** I can serve customers efficiently during peak hours

**Acceptance Criteria:**
- Handle multiple concurrent transactions
- No dependency on cellular network
- Fast device discovery (< 2 seconds)
- Visual confirmation for both parties

### US-4: Transaction Reconciliation
**As a** PaySetu user  
**I want** my offline transactions to sync automatically when I'm back online  
**So that** my account balance and history are always accurate

**Acceptance Criteria:**
- Automatic sync when connectivity detected
- Encrypted transmission to cloud
- Conflict resolution for duplicate entries
- Transaction integrity verification

## Key Features

### F-1: Instant P2P Connectivity
- Powered by Google Nearby Connections API
- Automatic device discovery via Bluetooth
- High-speed data transfer via Wi-Fi Direct
- Fallback to Bluetooth Low Energy (BLE)

### F-2: Hardware-Level Security
- Android Keystore System integration
- Trusted Execution Environment (TEE) for key storage
- Cryptographic operations isolated from main OS
- Protection against device compromise

### F-3: Encrypted Offline Ledger
- SQLCipher with 256-bit AES encryption
- Local transaction log on device
- Tamper-proof storage
- Efficient query performance

### F-4: Visual Transaction Confirmation
- Instant "Done Deal" status display
- QR code verification option
- Transaction receipt generation
- Fraud prevention through mutual confirmation

### F-5: Cloud Synchronization
- Automatic sync when online
- AWS Lambda for reconciliation logic
- Amazon DynamoDB for global ledger
- Conflict resolution algorithms

## Technical Architecture

### System Components

#### Frontend Layer
- **Platform:** Android (primary), iOS (future)
- **UI Framework:** Native Android SDK
- **Offline-First Design:** Local data persistence

#### Communication Layer
- **Primary:** Google Nearby Connections API
- **Transport:** Wi-Fi Direct (high-speed), Bluetooth (fallback)
- **Discovery:** Bluetooth Low Energy advertising
- **Range:** Up to 100 meters (Wi-Fi Direct)

#### Security Layer
- **Key Management:** Android Keystore System
- **Encryption:** AES-256 for data at rest
- **Transport Security:** TLS 1.3 for cloud sync
- **Authentication:** Hardware-backed cryptographic signatures

#### Data Layer
- **Local Database:** SQLCipher + SQLite
- **Schema:** Transactions, Users, Sync Queue
- **Encryption:** Database-level encryption
- **Backup:** Encrypted cloud backup

#### Backend Layer (Cloud Sync)
- **Compute:** AWS Lambda (serverless)
- **Database:** Amazon DynamoDB
- **Storage:** Amazon S3 (encrypted backups)
- **API:** REST API for mobile clients

### Data Flow

#### Offline Transaction Flow
1. **Discovery Phase**
   - Payer opens PaySetu app
   - App broadcasts availability via Bluetooth
   - Vendor device discovered automatically

2. **Connection Phase**
   - Secure Wi-Fi Direct connection established
   - Cryptographic handshake performed
   - Session keys exchanged

3. **Transaction Phase**
   - Payer initiates payment with amount
   - Transaction signed with hardware key
   - Credits transferred via P2P channel
   - Both ledgers updated locally

4. **Confirmation Phase**
   - Visual confirmation on both devices
   - Transaction receipt generated
   - Connection terminated

#### Online Sync Flow
1. **Network Detection**
   - App detects internet connectivity
   - Checks for pending transactions

2. **Sync Preparation**
   - Encrypts transaction batch
   - Generates sync payload
   - Signs with device key

3. **Cloud Reconciliation**
   - Lambda function validates transactions
   - DynamoDB updated with new records
   - Conflicts resolved (if any)

4. **Confirmation**
   - Sync status updated on device
   - User notified of successful sync

## Technology Stack

### Mobile Development
- **Language:** Kotlin (Android)
- **Min SDK:** Android 8.0 (API 26)
- **Target SDK:** Android 14 (API 34)

### Connectivity
- Google Nearby Connections API 18.0+
- Wi-Fi Direct (Android Wi-Fi P2P)
- Bluetooth Low Energy 5.0+

### Security
- Android Keystore System
- Trusted Execution Environment (TEE)
- SQLCipher 4.5+
- AES-256-GCM encryption

### Backend (AWS)
- AWS Lambda (Node.js 20.x runtime)
- Amazon DynamoDB (on-demand capacity)
- Amazon S3 (encrypted storage)
- AWS API Gateway (REST API)
- AWS CloudWatch (monitoring)

### Development Tools
- Android Studio Hedgehog+
- Gradle 8.2+
- Git for version control

## Unique Selling Propositions (USPs)

### 1. Zero-Internet Dependency
Complete transaction capability without any network connectivity (4G/5G/Wi-Fi/cellular data).

### 2. Hardware-Level Security
Cryptographic operations performed in isolated TEE, protecting against software-level attacks.

### 3. High-Speed P2P
Instant device discovery and settlement using Wi-Fi Direct (up to 250 Mbps theoretical).

### 4. Seamless Sync
Automatic reconciliation when back online without user intervention.

### 5. Inclusive Design
Works in shadow zones where traditional payment systems fail, enabling financial inclusion.

## Competitive Analysis

### vs. Traditional UPI/Net Banking
- **PaySetu:** Works offline, instant settlement
- **Traditional:** Requires server connection, fails in dead zones

### vs. SMS-Based Banking
- **PaySetu:** Free, fast, handles high traffic
- **Traditional:** Carrier charges, slow, unreliable in congestion

### vs. NFC Payments
- **PaySetu:** Longer range (100m), works without POS terminal
- **Traditional:** Requires physical proximity (< 4cm), needs merchant hardware

### vs. Blockchain/Crypto Wallets
- **PaySetu:** Fast, low overhead, fiat-based
- **Traditional:** Slow confirmation, high fees, volatile value

## Success Metrics

### Technical Metrics
- Transaction completion time: < 5 seconds
- Device discovery time: < 2 seconds
- Sync success rate: > 99%
- Encryption overhead: < 100ms

### Business Metrics
- User adoption in shadow zones
- Transaction volume growth
- Vendor onboarding rate
- Customer satisfaction score

### Impact Metrics
- Communities served in underconnected areas
- Reduction in cash dependency
- Financial inclusion improvement
- Transaction success rate vs. traditional methods

## Implementation Phases

### Phase 1: MVP (Minimum Viable Product)
- Basic P2P connectivity (Bluetooth only)
- Simple transaction flow
- Local encrypted storage
- Manual sync trigger

### Phase 2: Enhanced Connectivity
- Wi-Fi Direct integration
- Automatic device discovery
- Improved range and speed
- Better error handling

### Phase 3: Cloud Integration
- AWS backend setup
- Automatic sync when online
- Transaction reconciliation
- Conflict resolution

### Phase 4: Security Hardening
- TEE integration
- Hardware key storage
- Advanced fraud detection
- Audit logging

### Phase 5: Scale & Polish
- Multi-device support
- iOS version
- Advanced analytics
- Merchant dashboard

## Constraints & Considerations

### Technical Constraints
- Android 8.0+ required for Nearby Connections API
- Device must support Wi-Fi Direct or Bluetooth
- Minimum 100MB storage for app and ledger

### Security Considerations
- Device loss/theft mitigation (PIN/biometric lock)
- Transaction limits for offline mode
- Fraud detection algorithms
- Regular security audits

### Regulatory Compliance
- Financial transaction regulations
- Data privacy laws (GDPR, local equivalents)
- KYC requirements for larger transactions
- Anti-money laundering (AML) compliance

### Scalability Considerations
- Local ledger size management
- Sync queue optimization
- Cloud infrastructure auto-scaling
- Database sharding strategy

## Risk Assessment

### High Priority Risks
1. **Security Breach:** Mitigation via TEE and encryption
2. **Transaction Disputes:** Mitigation via cryptographic receipts
3. **Sync Failures:** Mitigation via retry logic and conflict resolution

### Medium Priority Risks
1. **Device Compatibility:** Mitigation via fallback protocols
2. **User Adoption:** Mitigation via education and incentives
3. **Regulatory Changes:** Mitigation via compliance monitoring

### Low Priority Risks
1. **Technology Obsolescence:** Mitigation via modular architecture
2. **Competition:** Mitigation via continuous innovation

## Future Enhancements

- AI-based fraud detection
- Multi-currency support
- Integration with existing UPI ecosystem
- Merchant analytics dashboard
- Loyalty program integration
- Voice-based transactions for accessibility
- Offline bill splitting
- Recurring payment support

## Appendix

### Glossary
- **Shadow Zone:** Area with poor or no internet connectivity
- **TEE:** Trusted Execution Environment - isolated secure area of processor
- **P2P:** Peer-to-Peer communication without intermediary server
- **SQLCipher:** Encrypted SQLite database extension

### References
- Google Nearby Connections API Documentation
- Android Keystore System Guide
- Wi-Fi Direct Technical Specification
- AWS Lambda Best Practices

---

**Document Status:** Ready for Design Phase  
**Next Steps:** Create design.md with detailed technical specifications and UI/UX mockups
