# PayMesh — Offline Mesh-Based Payment System

> A secure, distributed payment system designed to transfer payment requests across a device-to-device mesh when direct internet connectivity is unavailable.

## 📌 Overview

**PayMesh** is a Spring Boot-based payment system that explores how digital transactions can be delivered in environments with limited or no internet connectivity.

The system allows a payment request to be converted into an encrypted packet and propagated through a network of nearby devices. The packet can travel across multiple devices until it reaches an internet-connected bridge, which forwards the transaction to the backend for validation and settlement.

The project focuses on building a reliable payment pipeline while addressing important distributed-system challenges such as:

* Secure communication over untrusted devices
* Duplicate transaction processing
* Replay protection
* Transaction integrity
* Eventual settlement
* Concurrent transaction handling

> **Note:** This project is a simulation of an offline mesh payment architecture and does not connect to any real banking or UPI infrastructure.

---

## ✨ Features

### 🔐 End-to-End Encrypted Transactions

Payment information is encrypted before entering the mesh network.

The system uses:

* **RSA-OAEP** for asymmetric encryption
* **AES-256-GCM** for payment payload encryption
* Authentication tags to detect tampered packets

Intermediate devices only handle encrypted transaction packets and cannot access the underlying payment information.

---

### 📡 Offline Mesh Propagation

A payment does not require the sender to have an active internet connection.

Instead:

```text
Sender
   ↓
Device A
   ↓
Device B
   ↓
Device C
   ↓
Internet Bridge
   ↓
Backend
```

The project contains a software-based mesh simulator that models this device-to-device propagation.

Each packet contains a **TTL (Time To Live)** value that limits the number of hops through the network.

---

### 🔄 Idempotent Payment Processing

A transaction may reach the backend multiple times through different bridge devices.

To prevent duplicate settlements, PayMesh generates a SHA-256 hash of the encrypted transaction payload.

```text
Incoming Packet
      ↓
Generate Hash
      ↓
Idempotency Check
      ↓
Already Processed?
   ↙          ↘
 YES           NO
  ↓             ↓
Reject        Process
Duplicate        ↓
              Settle
```

This ensures that the same payment is settled only once.

---

### 🛡️ Replay & Tamper Protection

The system validates transactions before allowing them to reach the settlement layer.

Protection includes:

* Unique transaction nonce
* Timestamp validation
* Encrypted transaction payload
* AES-GCM authentication
* Duplicate packet detection

A modified encrypted payload fails authentication during decryption and is rejected.

---

### 💳 Atomic Settlement

Successful payments are processed inside a database transaction.

The settlement operation performs:

1. Sender balance validation
2. Sender debit
3. Receiver credit
4. Transaction ledger insertion

The operations are executed atomically to maintain consistency.

---

## 🏗️ System Architecture

```text
                    OFFLINE ENVIRONMENT

┌───────────────┐
│ Sender Device │
└───────┬───────┘
        │
        │ Encrypted Payment
        ▼
┌───────────────┐
│   Mesh Node   │
└───────┬───────┘
        │
        │ Forward
        ▼
┌───────────────┐
│   Mesh Node   │
└───────┬───────┘
        │
        │ Forward
        ▼
┌────────────────┐
│  Bridge Device │
│  Internet ✓    │
└────────┬───────┘
         │
         │ HTTPS
         ▼

              ONLINE ENVIRONMENT

┌──────────────────────────────────┐
│          Spring Boot API         │
│                                  │
│  ┌────────────────────────────┐  │
│  │ Packet Validation           │  │
│  └─────────────┬──────────────┘  │
│                ↓                 │
│  ┌────────────────────────────┐  │
│  │ Idempotency Check          │  │
│  └─────────────┬──────────────┘  │
│                ↓                 │
│  ┌────────────────────────────┐  │
│  │ Decryption & Validation    │  │
│  └─────────────┬──────────────┘  │
│                ↓                 │
│  ┌────────────────────────────┐  │
│  │ Transaction Settlement     │  │
│  └─────────────┬──────────────┘  │
│                ↓                 │
│  ┌────────────────────────────┐  │
│  │ Transaction Ledger         │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
```

---

## 🔄 Transaction Flow

### Step 1 — Create Payment

A payment request is created with information such as:

* Sender
* Receiver
* Amount
* Nonce
* Timestamp

The payment data is encrypted before being introduced into the mesh.

### Step 2 — Create Mesh Packet

The encrypted payment is wrapped inside a packet containing routing information.

```text
MeshPacket
├── packetId
├── ttl
├── createdAt
└── ciphertext
```

### Step 3 — Propagate Through Mesh

Mesh nodes exchange packets with nearby nodes.

Every hop decreases the packet TTL.

This allows the system to simulate transaction propagation across an offline environment.

### Step 4 — Bridge Upload

When a mesh node with internet connectivity receives the packet, it forwards the encrypted transaction to the backend.

### Step 5 — Backend Validation

The backend performs:

```text
Packet Received
      ↓
SHA-256 Hash
      ↓
Idempotency Check
      ↓
Decrypt Payload
      ↓
Validate Timestamp
      ↓
Validate Transaction
      ↓
Settlement
```

### Step 6 — Settlement

Once validated, the sender's account is debited, the receiver's account is credited, and the transaction is added to the ledger.

---

## 🔐 Security Architecture

### Hybrid Encryption

The project uses hybrid encryption to combine the security of RSA with the performance of AES.

```text
Payment Payload
      │
      ▼
 Generate AES Key
      │
      ├───────────────┐
      ▼               ▼
AES-256-GCM       RSA-OAEP
Encryption        Key Encryption
      │               │
      └───────┬───────┘
              ▼
       Encrypted Packet
```

AES-GCM also provides authentication, allowing the backend to detect modifications to the encrypted payload.

---

## 🔄 Handling Duplicate Requests

Consider a situation where multiple bridge devices receive the same payment:

```text
                 ┌── Bridge A ──┐
                 │              │
Sender → Mesh ───┼── Bridge B ──┼──→ Backend
                 │              │
                 └── Bridge C ──┘
```

All three bridges may attempt to submit the same transaction.

The backend calculates a hash of the encrypted payload and uses it to identify previously processed transactions.

Only one request is allowed to proceed to settlement.

The remaining requests are rejected as duplicates.

---

## 🧪 Testing

The project includes tests for the critical parts of the payment pipeline.

### Encryption Test

Validates encryption and decryption of payment data.

### Tamper Detection Test

Modifies an encrypted transaction and verifies that the backend rejects it.

### Concurrent Delivery Test

Simulates multiple bridge devices delivering the same transaction concurrently and verifies that the payment is settled exactly once.

---

## 🛠️ Tech Stack

| Technology        | Usage                         |
| ----------------- | ----------------------------- |
| Java 17           | Core application              |
| Spring Boot       | Backend & REST APIs           |
| Spring Data JPA   | Persistence layer             |
| H2 Database       | Development database          |
| Maven             | Build & dependency management |
| RSA-OAEP          | Key encryption                |
| AES-256-GCM       | Data encryption               |
| JUnit             | Automated testing             |
| HTML / JavaScript | Dashboard                     |

---

## 📂 Project Structure

```text
paymesh/
│
├── pom.xml
├── README.md
│
└── src/
    ├── main/
    │   ├── java/
    │   │   └── com/paymesh/
    │   │       ├── model/
    │   │       ├── crypto/
    │   │       ├── service/
    │   │       ├── controller/
    │   │       └── config/
    │   │
    │   └── resources/
    │       ├── application.properties
    │       └── templates/
    │           └── dashboard.html
    │
    └── test/
        └── java/
```

---

## 🚀 Getting Started

### Prerequisites

Make sure the following are installed:

* Java 17 or newer
* Git

### Clone the Repository

```bash
git clone <your-repository-url>
cd paymesh
```

### Run the Application

#### Windows

```bash
mvnw.cmd spring-boot:run
```

#### Linux / macOS

```bash
./mvnw spring-boot:run
```

The application will be available at:

```text
http://localhost:8080
```

---

## 🧪 Running Tests

### Windows

```bash
mvnw.cmd test
```

### Linux / macOS

```bash
./mvnw test
```

---

## 📊 Dashboard

The project includes an interactive dashboard that allows the complete payment lifecycle to be observed.

The dashboard provides visibility into:

* Mesh device states
* Payment creation
* Packet propagation
* Bridge uploads
* Account balances
* Transaction history
* Settlement results

---

## ⚠️ Limitations

This implementation is designed for learning and experimentation.

Currently:

* Mesh communication is simulated
* H2 is used for persistence
* Idempotency state is maintained in application memory
* Bridge authentication is simplified
* No real banking infrastructure is connected
* No real UPI transactions are performed
* Offline balance verification is simplified

These components can be replaced with production-grade infrastructure as the system evolves.

---

## 🔮 Future Improvements

Potential improvements include:

* PostgreSQL for persistent storage
* Redis-based distributed idempotency
* JWT-based authentication
* Secure bridge-node authentication
* Docker & Docker Compose
* Real Bluetooth/BLE communication
* Mobile Android client
* Rate limiting
* Distributed message queues
* Monitoring and structured logging
* CI/CD pipeline with GitHub Actions
* Improved mesh routing algorithms

---

## 🎯 Learning Outcomes

This project provides practical experience with:

* Spring Boot application architecture
* REST API development
* Distributed system concepts
* Cryptographic primitives
* Idempotency
* Concurrent request processing
* Database transactions
* Eventual consistency
* Secure data transmission
* Failure handling

---

## 👨‍💻 Author

**Bhaskar Vikas Bajaj**

A backend-focused project exploring distributed systems, secure payment processing, and Spring Boot application development.
