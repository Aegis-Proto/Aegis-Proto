# Aegis Secure Communication Protocol (ASCP) v1.3 Specification

## 1. SYSTEM ARCHITECTURE AND TRUST MODEL

### 1.1 Overview
The Aegis Secure Communication Protocol (ASCP) is a federated-ready, client-server communication protocol designed for high security and extensibility. The system operates on a zero-trust server model regarding message content. The server acts as a stateful message router and metadata store but possesses no cryptographic capability to decrypt user content.

### 1.2 Trust Boundaries
Trust is established solely between client devices belonging to a user and between users participating in a conversation. The server is considered hostile regarding data content but trusted for availability and routing logic. All sensitive data must be encrypted prior to leaving the client boundary.

### 1.3 Component Roles
- **Backend Core:** Handles authentication, routing, presence, and persistent storage of encrypted blobs. Implemented in Rust or Go.
- **Frontend Client:** Handles key management, encryption/decryption, rendering, and plugin execution. Implemented in TypeScript/React with WASM modules.
- **Plugin Sidecars:** External processes extending backend functionality via gRPC.
- **WASM Plugins:** Sandboxed modules extending frontend functionality.



## 2. CRYPTOGRAPHIC SPECIFICATION

### 2.1 Algorithm Suite
- **Asymmetric Encryption:** X25519 for key exchange. Ed25519 for digital signatures.
- **Symmetric Encryption:** AES-256-GCM for message content and database storage. ChaCha20-Poly1305 permitted for mobile clients lacking AES-NI.
- **Key Derivation:** Argon2id for password-based key derivation (local DB). HKDF-SHA256 for session key derivation.
- **Hashing:** BLAKE3 for content addressing and integrity checks.

### 2.2 Key Hierarchy
- **Identity Key (IK):** Long-term static key pair. Used to sign device keys. Stored in secure enclave where available.
- **Device Key (DK):** Unique key pair per device. Signed by IK.
- **PreKeys:** Batch of one-time use public keys uploaded to the server for asynchronous session initiation (X3DH protocol).
- **Session Key (SK):** Derived via Double Ratchet protocol for 1:1 conversations.
- **Sender Key (SenderChain):** Used for group channels. Rotated upon membership change.

### 2.3 Message Encryption Flow
1. Client generates a random message key (MK).
2. Content (text, markdown, metadata) is serialized to FlatBuffers.
3. Payload is encrypted using AES-256-GCM with MK.
4. MK is encrypted for each recipient's current Session Key.
5. Final packet contains encrypted payload and encrypted keys.

### 2.4 Reaction Privacy Model
To prevent metadata leakage regarding user activity, reactions are not stored as distinct server-side relational entities linked to user IDs.
1. Reactions are aggregated into a ReactionBundle object.
2. The Bundle is encrypted using the Channel Session Key.
3. The server stores only the encrypted blob and a version vector.
4. Server cannot determine which user added which emoji or the total count without decrypting.
5. Clients download the bundle, decrypt, and render counts locally.
6. Limit: Client-side enforcement of maximum 10 unique emoji types per message. Server validates blob size only.

### 2.5 Traffic Analysis Resistance
All transport payloads must be padded to multiples of 1024 bytes. This obscures message length and type differentiation from network observers. TLS 1.3 is mandatory for all transport layers to protect IP metadata and routing information.



## 3. TRANSPORT AND SERIALIZATION

### 3.1 Serialization Format
Wire format is Binary FlatBuffers. JSON is not permitted on the wire for message traffic to reduce parsing overhead and memory allocation. API documentation schemas are provided in OpenAPI format for logical understanding, but implementation must use generated FlatBuffers code.

### 3.2 Transport Protocols
- **Primary:** WebSocket (WSS). Full-duplex communication.
- **Fallback:** HTTP Long-Polling (HTTPS). Used when WebSocket connections are blocked by firewalls.
- **Switching Logic:** Client attempts WSS. On failure or persistent disconnect, switches to HTTP polling with exponential backoff. State synchronization is handled via cursor tokens, making transport switching transparent.

### 3.3 Packet Structure
Root Table: `Packet`
- `id`: string (UUID v4)
- `type`: enum (Message, Reaction, KeyUpload, SyncRequest)
- `timestamp`: int64 (Unix Epoch ms)
- `routing`: table (channel_id, thread_id) - Encrypted opaque hashes
- `payload`: vector (ubyte) - Encrypted content
- `signature`: vector (ubyte) - Ed25519 signature of payload
- `padding`: vector (ubyte) - Zeroes to reach 1024 multiple



## 4. API SPECIFICATION (LOGICAL VIEW)

*Note: While the wire format is FlatBuffers, the logical API structure follows RESTful principles for resource management.*

### 4.1 Authentication
**POST /api/v1/auth/register**
- **Request:** `username`, `password_hash`, `device_key_public`
- **Response:** `user_id`, `identity_key_signature`, `refresh_token`

**POST /api/v1/auth/login**
- **Request:** `username`, `password_hash`
- **Response:** `access_token`, `encrypted_private_key_bundle` (if backup enabled)

### 4.2 Synchronization
**GET /api/v1/sync**
- **Parameters:** `cursor` (string), `limit` (int)
- **Description:** Retrieves all events since the provided cursor.
- **Response:** List of Packet objects, `next_cursor` string.
- **Behavior:** If using HTTP polling, this request blocks until new events arrive or timeout (30s).

### 4.3 Message Transmission
**POST /api/v1/send**
- **Body:** Packet (FlatBuffer)
- **Response:** 202 Accepted, `event_id`
- **Note:** Server does not decrypt body. Validates signature and structure only.

### 4.4 Media Upload
**POST /api/v1/media/upload**
- **Headers:** `Content-Type`, `X-File-Hash`
- **Body:** Binary stream (encrypted file content)
- **Response:** `media_id`, `url`
- **Note:** URL is temporary signed link. Content remains encrypted at rest.

### 4.5 Key Management
**POST /api/v1/keys/upload**
- **Body:** List of PreKeys (public)
- **Description:** Uploads one-time keys for X3DH session establishment.

**GET /api/v1/keys/download/{user_id}**
- **Response:** List of available PreKeys for target user.



## 5. DATA STORAGE SPECIFICATION

### 5.1 Server Storage
- **Database:** PostgreSQL or ScyllaDB.
- **Encryption:** Data at rest encryption (TDE) enabled at disk level.
- **Schema Minimization:** Server stores only opaque blobs. No plaintext message content.
- **Tables:**
    - `users`: id, username_hash, password_hash, identity_key_public
    - `devices`: id, user_id, device_key_public, last_seen
    - `channels`: id, creator_id, encrypted_metadata_blob
    - `messages`: id, channel_id, thread_id, sender_device_id, encrypted_blob, timestamp
    - `reactions`: message_id, encrypted_bundle_blob, version
    - `cursors`: user_id, channel_id, last_event_id

### 5.2 Client Storage
- **Database:** SQLite with SQLCipher.
- **Encryption Key:** Derived from user password via Argon2id. Key stored in RAM only.
- **Schema:**
    - `local_messages`: id, channel_id, decrypted_content_cache, status (sent/pending/failed)
    - `sessions`: user_id, device_id, ratchet_state (encrypted)
    - `media_cache`: id, local_path, expiry_timestamp
- **Security Requirement:** Database file must be unreadable without deriving the key from user password or biometric unlock wrapper.



## 6. PLUGIN ARCHITECTURE

### 6.1 Frontend Plugins (WASM)
- **Runtime:** WebAssembly sandbox within the client application.
- **Interface:** JavaScript/TypeScript SDK exposing limited API.
- **Capabilities:**
    - **UI Components:** Inject React/Vue components into chat view.
    - **Message Processors:** Transform message content before rendering (e.g., custom markdown).
    - **Network Hooks:** Intercept outgoing packets (cannot modify encryption keys).
- **Security:** WASM modules cannot access the cryptographic key store directly. They must request encryption/decryption services via the host API, which enforces policy.
- **Manifest:** `plugin.json` defining name, version, permissions, `entry_point.wasm`.

### 6.2 Backend Plugins (Sidecars)
- **Architecture:** External processes communicating with Core via gRPC over Unix Domain Sockets or localhost TLS.
- **Integration Points:**
    - **Event Hooks:** Subscribe to message events (metadata only, content encrypted).
    - **Command Handlers:** Register slash commands (/plugin_command).
    - **Storage:** Access to dedicated plugin namespace in database.
- **Security:** Plugins run as separate OS users with restricted permissions. Cannot access master key material.
- **Manifest:** `plugin.yaml` defining gRPC service definition, required scopes.

### 6.3 Plugin Distribution
Plugins are signed by developers. Client verifies signature against trusted registry before loading. Enterprise deployments may allow unsigned internal plugins.



## 7. BOT INTEGRATION

### 7.1 Bot Identity
Bots are specialized user accounts with `is_bot` flag set. Bots possess Identity Keys and Device Keys like standard users. Bots participate in E2EE conversations. To decrypt messages, the bot's identity key must be verified by a human admin in the channel.

### 7.2 Authentication Flow
- OAuth 2.0 Authorization Code Grant.
- User authorizes bot to access specific Workspaces.
- Bot receives access token for API calls.
- For E2EE rooms: Bot must be invited. Bot client must manage its own key store to decrypt messages addressed to the channel.

### 7.3 Capabilities and Restrictions
- Bots cannot initiate E2EE sessions with users unless explicitly configured as trusted services.
- Bots have rate limits enforced by server.
- Bots can upload media and send messages via standard /send API.
- Bots cannot access reaction bundles unless they have the channel session key (i.e., are members).

### 7.4 Bot SDK
Provided in Python, Go, and TypeScript. Handles session management, encryption logic, and webhook reception automatically.



## 8. PROJECT DIRECTORY STRUCTURE
- **Backend:** Compiled to static binary. Docker container for deployment.
- **Frontend:** Compiled to static assets. Served via CDN or bundled with desktop app (Electron/Tauri).
- **Database Migrations:** Versioned SQL files managed by core service on startup.



## 9. INTERACTION SPECIFICATION (BACKEND-FRONTEND)

### 9.1 Session Initialization
1. Client sends login credentials.
2. Server validates and returns access token.
3. Client downloads PreKeys for contacts.
4. Client establishes WebSocket connection.
5. Client sends initial SyncRequest with cursor=0.
6. Server streams historical events (encrypted blobs).
7. Client decrypts and stores in local SQLCipher DB.

### 9.2 Message Sending
1. User types message.
2. Client encrypts content with Session Key.
3. Client pads payload to 1024 bytes.
4. Client sends via WebSocket.
5. Server acknowledges receipt (ACK).
6. Server broadcasts to other recipients.
7. Recipients decrypt and store.
8. If WebSocket fails, Client queues message in local DB Outbox.
9. Client retries via HTTP Polling POST /send when connection restored.

### 9.3 Plugin Loading
1. Client starts.
2. Client checks local plugin registry.
3. For each enabled plugin, Client loads WASM module.
4. Client calls plugin.init(context) where context contains limited API.
5. Plugin registers UI hooks or message processors.
6. Backend plugins are discovered via service discovery on server start.



## 10. SECURITY COMPLIANCE REQUIREMENTS

### 10.1 Key Storage
- Private keys must never be written to disk in plaintext.
- Use OS-specific secure storage (Keychain, Keystore, Secret Management Service).
- Memory containing keys must be zeroed after use.

### 10.2 Code Integrity
- All client binaries must be code-signed.
- Plugins must be verified via digital signature before execution.
- Supply chain security: Dependencies must be locked and audited.

### 10.3 Logging
- Server logs must not contain message content or unhashed user IDs.
- Correlation IDs used for debugging must be rotated frequently.
- Client logs must be disabled in production or encrypted locally.

### 10.4 Auditability
- Crypto implementation must be open source for review.
- Protocol specifications are public.
- Server code may be proprietary but must provide verifiable builds.


END OF SPECIFICATION
