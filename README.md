# 📡 UPI Offline Mesh — P2P Mesh-Routed Payment Simulator

A Spring Boot & Java 17 backend and interactive real-time visualizer that demonstrates **offline UPI payments routed through a Bluetooth-style peer-to-peer (P2P) mesh network**.

Imagine you are in a basement with zero internet connectivity. You want to send your friend $500. Your phone cryptographically wraps and encrypts the payment instruction, then broadcasts it to nearby devices. The packet hops device-to-device across a local Bluetooth mesh. Eventually, *some* device in the mesh walks outside, gets cellular coverage (4G/5G), and uploads it to the backend server. The backend decrypts, deduplicates, verifies freshness, and settles the payment.

This repository implements the **sovereign backend processing pipeline**, an **in-memory database ledger**, and a **software simulator** to demo the entire multi-hop flow on a single dashboard.

---

## 🚀 Key Visual & Feature Upgrade (New Front End)

The project includes an **interactive premium mission-control dashboard** served directly at `/` (via Spring Thymeleaf). 

* **Live SVG Network Graph**: Visualizes nodes (`phone-alice`, `phone-bob`, `phone-carol`, `phone-dave`, and `phone-bridge`) placed on an animated cyber-grid.
* **Real-time Packet Animation**: Watch transactions travel as glowing packet pulses hopping between devices during gossip cycles, and watch bridge nodes upload packets to the central cloud.
* **Synthesized Audio Effects**: Uses the HTML5 Web Audio API to play sound indicators for payment injections, gossip transfers, double-chime settlements, and error alerts (toggable on/off).
* **Live System Log**: A stylized scrollable console logging the exact cryptographic details (RSA encryption, AES-GCM decryption, hash generation, and seen-cache claims) in real-time.
* **Dynamic Ledger & Balances**: Account balance cards flash green (credits) or red (debits) with number animations as settlements occur.

---

## 🛠 Tech Stack

* **Language**: Java 17 (Spring Boot 3.3.5)
* **Build System**: Maven (via included wrapper `mvnw`)
* **Persistence**: Spring Data JPA + H2 In-Memory Database (zero-setup SQLite-like memory footprint)
* **Frontend**: HTML5, Vanilla CSS3 (custom variables, keyframe animations, glassmorphism), and Vanilla JavaScript (SVG overlays, Web Audio API, Fetch API).

---

## 🏗 System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SENDER PHONE (Offline)                          │
│  PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }  │
│              │                                                          │
│              ▼ Encrypt with Server's RSA Public Key (Hybrid Scheme)     │
│   MeshPacket { packetId, ttl, createdAt, ciphertext }                   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ Bluetooth BLE Gossip
                                       ▼
        ┌─────────┐  hop   ┌─────────┐  hop   ┌─────────┐
        │stranger1│ ─────▶ │stranger2│ ─────▶ │ bridge  │ ◀── walks outside
        └─────────┘        └─────────┘        └────┬────┘     gets 4G
                                                   │
                                                   ▼ HTTPS POST
┌─────────────────────────────────────────────────────────────────────────┐
│                     SPRING BOOT BACKEND (This Project)                  │
│                                                                         │
│  /api/bridge/ingest                                                     │
│       │                                                                 │
│       ▼                                                                 │
│  [1] Hash Ciphertext (SHA-256)                                          │
│       │                                                                 │
│       ▼                                                                 │
│  [2] IdempotencyService.claim(hash)  ◀── Atomic ConcurrentHashMap       │
│       │                                  (Mimics Redis SETNX). Drops    │
│       │                                  concurrent duplicates early.   │
│       ▼                                                                 │
│  [3] HybridCryptoService.decrypt(ciphertext)                            │
│       │       (RSA-OAEP decrypts AES-256 key, AES-GCM decrypts payload  │
│       │        and verifies the integrity tag. Any tamper throws error) │
│       ▼                                                                 │
│  [4] Freshness Check: Verify signedAt is within last 24 hours           │
│       │                                                                 │
│       ▼                                                                 │
│  [5] SettlementService.settle()                                         │
│       @Transactional: Debit sender, credit receiver, write ledger.      │
│       Uses @Version on Account for optimistic locking defense.          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🔒 The Three Hard Problems (And How They Are Solved)

### 1. Untrusted Intermediaries (Privacy & Integrity)
Because random strangers carry transaction packets on their phones, they must not be able to read the amount or change details.
* **Solution**: **Hybrid Encryption (RSA-OAEP-SHA256 + AES-256-GCM)**. 
  1. The sender generates a fresh AES-256 key and IV.
  2. The payload JSON is encrypted with AES-256-GCM (Authenticated Encryption).
  3. The AES key is encrypted using the server's RSA-2048 public key.
  4. The packet contains `[Encrypted AES Key (256 bytes)] + [IV (12 bytes)] + [Ciphertext + Auth Tag]`.
  Since only the backend holds the RSA Private Key, intermediaries see only raw bytes. If they mutate even a single bit, the GCM integrity verification tag fails on decryption, and the backend rejects the packet.

### 2. The Duplicate Storm (Double Settlement prevention)
If three bridges carry the same gossip packet and walk into cellular range at the same second, they will simultaneously upload it to `/api/bridge/ingest`. If naively processed, the sender is debited three times.
* **Solution**: **Atomic seen-cache on the ciphertext hash**.
  * The server hashes the ciphertext (`SHA-256`) and tries to claim it inside an atomic cache using `ConcurrentHashMap.putIfAbsent(hash, timestamp)` (JVM equivalent of Redis `SET key value NX EX 86400`).
  * Only the first claimer proceeds to decrypt and settle. The duplicate requests are short-circuited as `DUPLICATE_DROPPED` within microseconds, avoiding expensive RSA decryption or database lock contention.
  * *Defense-in-depth*: The database table `transactions` has a unique index on `packet_hash`, rejecting duplicate insertion on the database layer.

### 3. Replay Attacks
An attacker could intercept a valid encrypted ciphertext and re-upload it days later to debit the sender's account again.
* **Solution**: **Nonces + Freshness TTL**.
  * The encrypted envelope contains a unique `nonce` (UUID) and a `signedAt` timestamp.
  * The server rejects any packet where `signedAt` is older than 24 hours.
  * If the attacker tries to replay a packet within the 24-hour window, the ciphertext is identical, meaning the seen-cache hash check will flag and drop it. They cannot modify `signedAt` because it is sealed inside the AES-GCM ciphertext.

---

## 📂 Code Layout & File Directory

```
upi-offline-mesh/
├── pom.xml                                  Spring Boot 3.3, Java 17 configuration
├── mvnw, mvnw.cmd                           Maven wrappers (runs without Maven installation)
├── README.md                                Recruiter guide & technical specifications
└── src/main/
    ├── resources/
    │   ├── application.properties           H2 database config & TTL settings
    │   └── templates/
    │       └── dashboard.html               Interactive HTML/CSS/JS frontend visualizer
    └── java/com/demo/upimesh/
        ├── UpiMeshApplication.java          Spring Boot Application Entry point
        │
        ├── model/                           ── Domain Entities & Repositories
        │   ├── Account.java                 JPA account schema with optimistic lock versioning
        │   ├── AccountRepository.java       Database interface for Account queries
        │   ├── Transaction.java             Ledger entity with unique index on packetHash
        │   ├── TransactionRepository.java   Database interface for Transaction logs
        │   ├── MeshPacket.java              Outer envelope wire format (unencrypted fields)
        │   └── PaymentInstruction.java      Decrypted payload format (sender/receiver/amount/nonce)
        │
        ├── crypto/                          ── Security Layer
        │   ├── ServerKeyHolder.java         Generates RSA-2048 keypair dynamically on startup
        │   └── HybridCryptoService.java     Hybrid RSA-OAEP + AES-GCM Encryption/Decryption
        │
        ├── service/                         ── Core Services
        │   ├── DemoService.java             Seeds test accounts and simulates offline sender phones
        │   ├── VirtualDevice.java           Simulated device holding packet queues
        │   ├── MeshSimulatorService.java    P2P BLE gossip routing simulator
        │   ├── IdempotencyService.java      ConcurrentHashMap cache that prevents double-spends
        │   ├── SettlementService.java       Transactional balance updates and ledger writes
        │   └── BridgeIngestionService.java  Main REST ingestion pipeline (verify -> decrypt -> settle)
        │
        └── controller/                      ── REST Controllers
            ├── ApiController.java           REST Endpoint definitions (/api/mesh/*, /api/bridge/*)
            └── DashboardController.java     Serves index page (templates/dashboard.html)
```

---

## 🏃 Local Setup & Run Instructions

### Prerequisites
* **JDK 17 or higher** installed. If you do not have it, install via Homebrew on Mac:
  ```bash
  brew install openjdk@17
  sudo ln -sfn /opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk
  ```
  Check by running `java -version`.

### Launching the Application
1. Navigate to the project directory:
   ```bash
   cd offlineUPI
   ```
2. Build and run the server:
   ```bash
   ./mvnw spring-boot:run
   ```
3. Open your browser to:
   👉 **[http://localhost:8080](http://localhost:8080)**

---

## 🧪 Running Automated Tests

A core test suite verifies the cryptographic and concurrency safety of the pipeline:
```bash
./mvnw test
```
Key tests in `IdempotencyConcurrencyTest`:
* `encryptDecryptRoundTrip`: Verifies RSA/AES-GCM encryption symmetry.
* `tamperedCiphertextIsRejected`: Flipping a single bit in the ciphertext forces the GCM tag check to fail, ensuring the request is flagged as `INVALID`.
* `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`: Fires 3 parallel threads delivering the identical packet simultaneously to replicate the concurrent duplicate storm. Asserts exactly one settles and two are dropped.

---

## ☁️ Railway Deployment Instructions

This app is configured to build on Railway automatically:
1. Log in to [Railway.app](https://railway.app).
2. Create a **New Project** -> **Deploy from GitHub repository**.
3. Select your `offlineUPI` repository.
4. Railway's Nixpacks builder will automatically detect Java 17 and Maven, build the fat jar, and launch it.
5. In settings, click **Generate Domain** to get a public URL for your simulator.

---

## ⚖️ Real-World Limitations
1. **Funds Verification**: Because the sender is offline, the receiver has no guarantee that the sender's account has funds. In real-world systems like *UPI Lite*, this is solved by using a hardware-backed secure element holding pre-funded wallets.
2. **Double Spends**: An offline user could double-spend their balance by sending payments to two separate people in different offline spots. The backend resolves this by only settling the packet that arrives first, rejecting the second.
