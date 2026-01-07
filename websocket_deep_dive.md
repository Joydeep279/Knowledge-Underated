# WebSocket Protocol - Complete Technical Documentation

## Table of Contents
1. [Introduction & Fundamentals](#introduction)
2. [Protocol Comparison](#comparison)
3. [The WebSocket Handshake (Detailed)](#handshake)
4. [Frame Structure & Binary Protocol](#framing)
5. [Masking Mechanism](#masking)
6. [Message Fragmentation](#fragmentation)
7. [Control Frames](#control-frames)
8. [Implementation Examples](#implementation)
9. [Security Considerations](#security)
10. [Performance & Best Practices](#best-practices)
11. [Troubleshooting](#troubleshooting)

---

## 1. Introduction & Fundamentals {#introduction}

### What is WebSocket?

WebSocket is a **full-duplex communication protocol** that operates over a single TCP connection. Standardized as **RFC 6455**, it provides a persistent connection between client and server, enabling real-time bidirectional data exchange.

### Key Characteristics

- **Persistent Connection**: Stays open after establishment, unlike HTTP's request-response cycle
- **Full-Duplex**: Both client and server can send data independently and simultaneously
- **Low Overhead**: Minimal frame header (2-14 bytes) compared to HTTP headers (hundreds of bytes)
- **Real-Time**: No polling needed; instant message delivery
- **Protocol Upgrade**: Starts as HTTP, upgrades to WebSocket

### URI Schemes

```
ws://example.com:80/path          # Unencrypted (Default port 80)
wss://example.com:443/path        # Encrypted via TLS (Default port 443)
```

**Critical**: Always use `wss://` in production for security.

---

## 2. Protocol Comparison {#comparison}

### HTTP vs WebSocket

| Feature | HTTP | WebSocket |
|---------|------|-----------|
| **Connection Type** | Request-Response, Short-lived | Persistent, Long-lived |
| **Communication** | Half-duplex (One direction at a time) | Full-duplex (Simultaneous) |
| **Overhead** | High (Headers repeated per request) | Low (Small frame headers) |
| **Server Push** | No (requires polling/SSE) | Yes (Native) |
| **Latency** | Higher (New connection per request) | Lower (Single persistent connection) |
| **Use Case** | Document retrieval, APIs | Real-time applications |

### When to Use WebSocket

**✅ Use WebSocket for:**
- Live chat applications
- Real-time notifications
- Multiplayer games
- Collaborative editing tools
- Live sports/stock tickers
- IoT device communication
- Live audio/video streaming metadata

**❌ Don't use WebSocket for:**
- Simple request-response APIs (Use REST)
- Infrequent updates (Use polling/SSE)
- File downloads (Use HTTP)
- SEO-critical content (Use HTTP)

---

## 3. The WebSocket Handshake (Detailed) {#handshake}

The handshake is a **one-time HTTP upgrade process** that transitions from HTTP to WebSocket protocol.

### Step-by-Step Handshake Process

#### Phase 1: Client Initiates Connection

The client sends an HTTP GET request with special headers:

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Extensions: permessage-deflate
```

**Header Breakdown:**

1. **`Upgrade: websocket`**
   - Signals intent to change protocols
   - Required for protocol upgrade

2. **`Connection: Upgrade`**
   - Confirms the connection should be upgraded
   - Prevents intermediary proxies from misinterpreting

3. **`Sec-WebSocket-Key`**
   - A **random 16-byte value** encoded in Base64
   - Generated uniquely for each handshake
   - Used to prevent cross-protocol attacks
   - Example generation (JavaScript):
   ```javascript
   const key = btoa(String.fromCharCode(...crypto.getRandomValues(new Uint8Array(16))));
   ```

4. **`Sec-WebSocket-Version: 13`**
   - **Must be 13** (RFC 6455 standard)
   - Server rejects other versions with HTTP 400

5. **`Origin`** (Browser-sent)
   - Identifies the script's origin
   - Server can validate to prevent CSRF attacks

6. **`Sec-WebSocket-Protocol`** (Optional)
   - Comma-separated list of sub-protocols
   - Examples: `chat`, `mqtt`, `stomp`, `json-rpc`

7. **`Sec-WebSocket-Extensions`** (Optional)
   - Compression, multiplexing, etc.
   - Example: `permessage-deflate` for compression

#### Phase 2: Server Validates and Responds

If valid, server responds with **HTTP 101 Switching Protocols**:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

**Critical Header: `Sec-WebSocket-Accept`**

This header **proves the server understood the WebSocket handshake**. It's computed using a cryptographic algorithm.

### The Magic UUID: `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`

This is a **fixed constant defined in RFC 6455**. It's concatenated with the client's key to prevent accidental WebSocket acceptance by non-WebSocket servers.

#### Acceptance Key Algorithm

```
Algorithm:
1. Take client's Sec-WebSocket-Key
2. Concatenate with Magic UUID
3. Compute SHA-1 hash
4. Base64 encode the result
```

**Example Implementation (Node.js):**

```javascript
const crypto = require('crypto');

function generateAcceptKey(clientKey) {
    const MAGIC_UUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
    const concatenated = clientKey + MAGIC_UUID;
    const sha1Hash = crypto.createHash('sha1').update(concatenated).digest();
    return sha1Hash.toString('base64');
}

// Example
const clientKey = 'dGhlIHNhbXBsZSBub25jZQ==';
const acceptKey = generateAcceptKey(clientKey);
console.log(acceptKey); // Output: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**Python Implementation:**

```python
import hashlib
import base64

def generate_accept_key(client_key):
    MAGIC_UUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11'
    concatenated = (client_key + MAGIC_UUID).encode('utf-8')
    sha1_hash = hashlib.sha1(concatenated).digest()
    return base64.b64encode(sha1_hash).decode('utf-8')

# Example
client_key = 'dGhlIHNhbXBsZSBub25jZQ=='
accept_key = generate_accept_key(client_key)
print(accept_key)  # Output: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

#### Phase 3: Connection Established

After receiving the 101 response:
- HTTP connection is **upgraded** to WebSocket
- TCP connection remains open
- Both sides can now send WebSocket frames
- No more HTTP semantics apply

---

## 4. Frame Structure & Binary Protocol {#framing}

After the handshake, all data is transmitted in **frames**. A message may consist of one or multiple frames.

### Complete Frame Structure (Byte-level)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               | Masking-key, if MASK set to 1 |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

### Field-by-Field Breakdown

#### Byte 0 (First Byte)

| Bit | Field | Description |
|-----|-------|-------------|
| 0 | **FIN** | **Final Fragment**: `1` = Last frame of message, `0` = More frames follow |
| 1-3 | **RSV1-3** | **Reserved**: Must be `0` unless extension negotiated |
| 4-7 | **Opcode** | **Frame Type**: Defines payload interpretation |

#### Byte 1 (Second Byte)

| Bit | Field | Description |
|-----|-------|-------------|
| 8 | **MASK** | **Masking Bit**: `1` = Payload is masked (Client→Server), `0` = Unmasked (Server→Client) |
| 9-15 | **Payload Len** | **Initial Length**: 0-125 = Actual length, 126 = Next 2 bytes, 127 = Next 8 bytes |

### Opcode Reference Table

| Opcode (Hex) | Opcode (Binary) | Frame Type | Description |
|--------------|-----------------|------------|-------------|
| `0x0` | `0000` | Continuation | Fragment of previous message |
| `0x1` | `0001` | Text | UTF-8 text data |
| `0x2` | `0010` | Binary | Raw binary data |
| `0x3-7` | `0011-0111` | Reserved | Reserved for future data frames |
| `0x8` | `1000` | Close | Connection close frame |
| `0x9` | `1001` | Ping | Heartbeat request |
| `0xA` | `1010` | Pong | Heartbeat response |
| `0xB-F` | `1011-1111` | Reserved | Reserved for future control frames |

### Payload Length Encoding

WebSocket uses a **variable-length encoding** for payload size:

#### Case 1: Small Payload (0-125 bytes)

```
Payload Len = Actual length
```

**Example**: 100-byte payload
```
Byte 1: 0x64 (binary: 01100100)
        ↑ MASK bit
         ↑↑↑↑↑↑↑ Length = 100
```

#### Case 2: Medium Payload (126-65,535 bytes)

```
Payload Len = 126
Next 2 bytes = Actual length (16-bit unsigned integer)
```

**Example**: 1,000-byte payload
```
Byte 1: 0xFE (126 in lower 7 bits)
Bytes 2-3: 0x03E8 (1000 in big-endian)
```

#### Case 3: Large Payload (> 65,535 bytes)

```
Payload Len = 127
Next 8 bytes = Actual length (64-bit unsigned integer)
```

**Example**: 100,000-byte payload
```
Byte 1: 0xFF (127 in lower 7 bits)
Bytes 2-9: 0x00000000000186A0 (100,000 in big-endian)
```

### Frame Examples

#### Example 1: Unmasked Text Frame (Server → Client)

**Message**: `"Hello"`

```
Byte 0: 0x81
  FIN=1 (Final frame)
  RSV=0
  Opcode=0x1 (Text)

Byte 1: 0x05
  MASK=0 (No masking)
  Payload Len=5

Bytes 2-6: 0x48656C6C6F ("Hello" in ASCII/UTF-8)

Complete Frame (Hex): 81 05 48 65 6C 6C 6F
```

#### Example 2: Masked Text Frame (Client → Server)

**Message**: `"Hello"`
**Masking Key**: `0x12345678`

```
Byte 0: 0x81 (FIN=1, Opcode=Text)
Byte 1: 0x85 (MASK=1, Len=5)
Bytes 2-5: 0x12345678 (Masking key)
Bytes 6-10: Masked payload

Masking calculation:
H (0x48) XOR 0x12 = 0x5A
e (0x65) XOR 0x34 = 0x51
l (0x6C) XOR 0x56 = 0x3A
l (0x6C) XOR 0x78 = 0x14
o (0x6F) XOR 0x12 = 0x7D

Complete Frame: 81 85 12 34 56 78 5A 51 3A 14 7D
```

#### Example 3: Ping Frame

```
Byte 0: 0x89 (FIN=1, Opcode=Ping)
Byte 1: 0x00 (No payload)

Complete Frame: 89 00
```

---

## 5. Masking Mechanism {#masking}

### Purpose of Masking

**Problem Prevented**: **Cache Poisoning Attack**

Without masking, intermediary proxies might interpret WebSocket frames as HTTP commands, enabling attackers to inject malicious cache entries.

**RFC 6455 Requirement**:
- **Client → Server**: MUST be masked
- **Server → Client**: MUST NOT be masked

### Masking Algorithm

```
masked_byte[i] = original_byte[i] XOR masking_key[i mod 4]
```

The **masking key** is a random 32-bit (4-byte) value.

### Implementation Example

**JavaScript (Client-side):**

```javascript
function maskPayload(payload, maskingKey) {
    const masked = new Uint8Array(payload.length);
    for (let i = 0; i < payload.length; i++) {
        masked[i] = payload[i] ^ maskingKey[i % 4];
    }
    return masked;
}

// Example
const payload = new TextEncoder().encode("Test");
const maskingKey = crypto.getRandomValues(new Uint8Array(4));
const masked = maskPayload(payload, maskingKey);
```

**Python:**

```python
def mask_payload(payload, masking_key):
    masked = bytearray()
    for i, byte in enumerate(payload):
        masked.append(byte ^ masking_key[i % 4])
    return bytes(masked)

# Example
payload = b"Test"
masking_key = bytes([0x12, 0x34, 0x56, 0x78])
masked = mask_payload(payload, masking_key)
```

### Unmasking (Server-side)

Since XOR is its own inverse:
```
original_byte[i] = masked_byte[i] XOR masking_key[i mod 4]
```

The same function can unmask!

---

## 6. Message Fragmentation {#fragmentation}

Fragmentation allows sending large messages in **multiple frames** without buffering the entire payload.

### Use Cases

1. **Unknown Total Size**: Streaming data where length isn't known upfront
2. **Memory Efficiency**: Avoid buffering massive payloads
3. **Interleaving**: Send control frames between data fragments

### Fragmentation Rules

1. **First Fragment**: `FIN=0`, Opcode = `0x1` (Text) or `0x2` (Binary)
2. **Middle Fragments**: `FIN=0`, Opcode = `0x0` (Continuation)
3. **Final Fragment**: `FIN=1`, Opcode = `0x0` (Continuation)

### Example: Sending "Hello World" in 3 Fragments

**Fragment 1**: `"Hello"`
```
FIN=0, Opcode=0x1 (Text), Payload="Hello"
Frame: 01 05 48 65 6C 6C 6F
```

**Fragment 2**: `" Wor"`
```
FIN=0, Opcode=0x0 (Continuation), Payload=" Wor"
Frame: 00 04 20 57 6F 72
```

**Fragment 3**: `"ld"`
```
FIN=1, Opcode=0x0 (Continuation), Payload="ld"
Frame: 80 02 6C 64
```

**Reassembled Message**: `"Hello World"`

### Important Notes

- **Cannot interleave data messages**: Once fragmentation starts, all fragments must arrive before starting a new message
- **Can interleave control frames**: Ping/Pong can be sent between fragments
- **Opcode only in first fragment**: Middle/final fragments use continuation opcode

---

## 7. Control Frames {#control-frames}

Control frames have **Opcode ≥ 0x8** and manage connection state.

### Characteristics

- **Must NOT be fragmented**: FIN must always be 1
- **Payload limit**: Maximum 125 bytes
- **Can be sent anytime**: Even between fragmented data frames

### Close Frame (0x8)

Initiates connection termination.

**Payload Structure**:
```
[2 bytes: Close Code] [Optional: UTF-8 Reason]
```

**Common Close Codes**:

| Code | Name | Meaning |
|------|------|---------|
| 1000 | Normal Closure | Successful completion |
| 1001 | Going Away | Server shutdown or browser navigation |
| 1002 | Protocol Error | Invalid frame received |
| 1003 | Unsupported Data | Received unacceptable data type |
| 1006 | Abnormal Closure | Connection lost without close frame |
| 1007 | Invalid Payload | Non-UTF-8 data in text frame |
| 1008 | Policy Violation | Generic policy violation |
| 1009 | Message Too Big | Payload exceeds limit |
| 1011 | Internal Error | Server encountered error |

**Example Close Frame**:
```
FIN=1, Opcode=0x8, Payload=1000 + "Goodbye"
Frame: 88 09 03 E8 47 6F 6F 64 62 79 65
       ↑  ↑  ↑     ↑
       |  |  |     └─ "Goodbye"
       |  |  └─ Close code 1000 (0x03E8)
       |  └─ Payload length (9 bytes)
       └─ FIN + Opcode
```

### Ping Frame (0x9) & Pong Frame (0xA)

**Purpose**: Keep-alive mechanism to detect dead connections.

**Flow**:
1. Sender sends Ping (optional payload)
2. Receiver responds with Pong (echoing payload)
3. If no Pong received within timeout → connection is dead

**Ping Frame Example**:
```
FIN=1, Opcode=0x9, Payload="ping"
Frame: 89 04 70 69 6E 67
```

**Corresponding Pong**:
```
FIN=1, Opcode=0xA, Payload="ping" (echoed)
Frame: 8A 04 70 69 6E 67
```

**Implementation (JavaScript)**:

```javascript
// Client sends ping
function sendPing(ws) {
    // Note: Browser WebSocket API doesn't expose Ping/Pong
    // This is typically done automatically
    // Server-side example with ws library:
    ws.ping('ping-data');
}

// Server handles pong
ws.on('pong', (data) => {
    console.log('Received pong:', data.toString());
});
```

---

## 8. Implementation Examples {#implementation}

### Client-Side (JavaScript/Browser)

```javascript
// Create WebSocket connection
const ws = new WebSocket('wss://example.com/socket');

// Connection opened
ws.addEventListener('open', (event) => {
    console.log('Connected to server');
    ws.send('Hello Server!');
});

// Receive messages
ws.addEventListener('message', (event) => {
    console.log('Message from server:', event.data);
    
    // Handle different data types
    if (typeof event.data === 'string') {
        console.log('Text message:', event.data);
    } else if (event.data instanceof Blob) {
        console.log('Binary message (Blob)');
    } else if (event.data instanceof ArrayBuffer) {
        console.log('Binary message (ArrayBuffer)');
    }
});

// Handle errors
ws.addEventListener('error', (error) => {
    console.error('WebSocket error:', error);
});

// Connection closed
ws.addEventListener('close', (event) => {
    console.log('Disconnected:', event.code, event.reason);
    if (event.wasClean) {
        console.log('Clean closure');
    } else {
        console.log('Connection lost');
    }
});

// Send different data types
ws.send('Text message');
ws.send(new Blob(['binary data']));
ws.send(new Uint8Array([1, 2, 3, 4]));
ws.send(JSON.stringify({ type: 'chat', message: 'Hello' }));

// Close connection
ws.close(1000, 'Normal closure');
```

### Server-Side (Node.js with `ws` library)

```javascript
const WebSocket = require('ws');

// Create WebSocket server
const wss = new WebSocket.Server({ port: 8080 });

// Handle new connections
wss.on('connection', (ws, req) => {
    console.log('New client connected from', req.socket.remoteAddress);
    
    // Send welcome message
    ws.send('Welcome to the server!');
    
    // Handle messages
    ws.on('message', (data, isBinary) => {
        console.log('Received:', isBinary ? 'Binary' : 'Text', data);
        
        // Echo back to client
        ws.send(data, { binary: isBinary });
        
        // Broadcast to all clients
        wss.clients.forEach((client) => {
            if (client !== ws && client.readyState === WebSocket.OPEN) {
                client.send(data, { binary: isBinary });
            }
        });
    });
    
    // Handle ping/pong
    ws.on('ping', (data) => {
        console.log('Received ping');
        ws.pong(data);
    });
    
    ws.on('pong', (data) => {
        console.log('Received pong');
    });
    
    // Handle errors
    ws.on('error', (error) => {
        console.error('WebSocket error:', error);
    });
    
    // Handle close
    ws.on('close', (code, reason) => {
        console.log('Client disconnected:', code, reason.toString());
    });
    
    // Set up heartbeat
    ws.isAlive = true;
    ws.on('pong', () => { ws.isAlive = true; });
});

// Heartbeat to detect dead connections
const interval = setInterval(() => {
    wss.clients.forEach((ws) => {
        if (ws.isAlive === false) {
            return ws.terminate();
        }
        ws.isAlive = false;
        ws.ping();
    });
}, 30000); // Every 30 seconds

wss.on('close', () => {
    clearInterval(interval);
});
```

### Python Server (with `websockets` library)

```python
import asyncio
import websockets

async def handler(websocket, path):
    print(f"Client connected from {websocket.remote_address}")
    
    try:
        # Send welcome message
        await websocket.send("Welcome to the server!")
        
        # Handle messages
        async for message in websocket:
            print(f"Received: {message}")
            
            # Echo back
            await websocket.send(f"Echo: {message}")
            
    except websockets.exceptions.ConnectionClosed:
        print("Client disconnected")

# Start server
async def main():
    async with websockets.serve(handler, "localhost", 8080):
        print("Server started on ws://localhost:8080")
        await asyncio.Future()  # Run forever

asyncio.run(main())
```

---

## 9. Security Considerations {#security}

### Always Use WSS in Production

```javascript
// ❌ INSECURE
const ws = new WebSocket('ws://example.com/socket');

// ✅ SECURE
const ws = new WebSocket('wss://example.com/socket');
```

### Validate Origin Header (Server)

```javascript
wss.on('connection', (ws, req) => {
    const origin = req.headers.origin;
    const allowedOrigins = ['https://example.com', 'https://app.example.com'];
    
    if (!allowedOrigins.includes(origin)) {
        ws.close(1008, 'Origin not allowed');
        return;
    }
    
    // Continue with connection...
});
```

### Authentication Strategies

#### Option 1: Token in URL

```javascript
const token = 'user-auth-token';
const ws = new WebSocket(`wss://example.com/socket?token=${token}`);
```

**Server validation**:
```javascript
wss.on('connection', (ws, req) => {
    const url = new URL(req.url, 'wss://example.com');
    const token = url.searchParams.get('token');
    
    if (!validateToken(token)) {
        ws.close(1008, 'Invalid token');
        return;
    }
});
```

#### Option 2: Authentication Message

```javascript
const ws = new WebSocket('wss://example.com/socket');

ws.addEventListener('open', () => {
    ws.send(JSON.stringify({ type: 'auth', token: 'user-token' }));
});
```

### Rate Limiting

```javascript
const connectionAttempts = new Map();

wss.on('connection', (ws, req) => {
    const ip = req.socket.remoteAddress;
    const attempts = connectionAttempts.get(ip) || 0;
    
    if (attempts > 10) {
        ws.close(1008, 'Too many connections');
        return;
    }
    
    connectionAttempts.set(ip, attempts + 1);
    
    // Clear after timeout
    setTimeout(() => {
        connectionAttempts.delete(ip);
    }, 60000);
});
```

### Input Validation

```javascript
ws.on('message', (data) => {
    try {
        const message = JSON.parse(data);
        
        // Validate structure
        if (!message.type || !message.payload) {
            throw new Error('Invalid message structure');
        }
        
        // Validate size
        if (data.length > 1024 * 1024) { // 1MB limit
            throw new Error('Message too large');
        }
        
        // Process message...
        
    } catch (error) {
        console.error('Invalid message:', error);
        ws.close(1003, 'Invalid data format');
    }
});
```

---

## 10. Performance & Best Practices {#best-practices}

### Connection Management

#### Implement Reconnection Logic

```javascript
class ReconnectingWebSocket {
    constructor(url, options = {}) {
        this.url = url;
        this.reconnectDelay = options.reconnectDelay || 1000;
        this.maxReconnectDelay = options.maxReconnectDelay || 30000;
        this.reconnectAttempts = 0;
        this.connect();
    }
    
    connect() {
        this.ws = new WebSocket(this.url);
        
        this.ws.onopen = () => {
            console.log('Connected');
            this.reconnectAttempts = 0;
        };
        
        this.ws.onclose = () => {
            console.log('Disconnected, reconnecting...');
            this.reconnect();
        };
        
        this.ws.onerror = (error) => {
            console.error('Error:', error);
        };
    }
    
    reconnect() {
        const delay = Math.min(
            this.reconnectDelay * Math.pow(2, this.reconnectAttempts),
            this.maxReconnectDelay
        );
        
        setTimeout(() => {
            this.reconnectAttempts++;
            this.connect();
        }, delay);
    }
    
    send(data) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(data);
        } else {
            console.warn('WebSocket not open');
        }
    }
}
```

#### Heartbeat/Keep-Alive

```javascript
let heartbeatInterval;

ws.addEventListener('open', () => {
    heartbeatInterval = setInterval(() => {
        if (ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify({ type: 'ping' }));
        }
    }, 30000); // Every 30 seconds
});

ws.addEventListener('close', () => {
    clearInterval(heartbeatInterval);
});
```

### Message Optimization

#### Batch Messages

```javascript
// ❌ Inefficient: Multiple sends
ws.send(JSON.stringify({ type: 'update', id: 1 }));
ws.send(JSON.stringify({ type: 'update', id: 2 }));
ws.send(JSON.stringify({ type: 'update', id: 3 }));

// ✅ Efficient: Single batched send
ws.send(JSON.stringify({
    type: 'batch',
    messages: [
        { type: 'update', id: 1 },
        { type: 'update', id: 2 },
        { type: 'update', id: 3 }
    ]
}));
```

#### Use Binary for Efficiency

```javascript
// Text: ~50 bytes
ws.send(JSON.stringify({ x: 100, y: 200, z: 300 }));

// Binary: 12 bytes
const buffer = new ArrayBuffer(12);
const view = new DataView(buffer);
view.setInt32(0, 100);  // x
view.setInt32(4, 200);  // y
view.setInt32(8, 300);  // z
ws.send(buffer);
```

#### Compression (permessage-deflate)

Most modern WebSocket implementations support automatic compression:

```javascript
// Server (ws library)
const wss = new WebSocket.Server({
    port: 8080,
    perMessageDeflate: {
        zlibDeflateOptions: {
            chunkSize: 1024,
            memLevel: 7,
            level: 3
        },
        zlibInflateOptions: {
            chunkSize: 10 * 1024
        },
        threshold: 1024 // Only compress messages > 1KB
    }
});
```

### Load Balancing

For horizontal scaling:

1. **Sticky Sessions**: Route same client to same server
2. **Redis Pub/Sub**: Share messages across servers
3. **Message Queue**: Decouple processing from connections

```javascript
// Example with Redis
const redis = require('redis');
const subscriber = redis.createClient();
const publisher = redis.createClient();

// Subscribe to channel
subscriber.subscribe('broadcast');

subscriber.on('message', (channel, message) => {
    // Broadcast to all connected clients on this server
    wss.clients.forEach((client) => {
        if (client.readyState === WebSocket.OPEN) {
            client.send(message);
        }
    });
});

// Publish message from any server
ws.on('message', (data) => {
    publisher.publish('broadcast', data);
});
```

---

## 11. Troubleshooting {#troubleshooting}

### Common Issues

#### Issue 1: "WebSocket connection failed"

**Causes**:
- Wrong protocol (ws vs wss)
- CORS/Origin blocking
- Firewall/proxy blocking
- Server not listening

**Solutions**:
```javascript
// Check protocol matches page
if (window.location.protocol === 'https:') {
    ws = new WebSocket('wss://example.com');
} else {
    ws = new WebSocket('ws://example.com');
}

// Verify server is running
// Check browser console for specific error
```

#### Issue 2: Connection drops randomly

**Causes**:
- No heartbeat/keep-alive
- Proxy timeout
- Resource exhaustion

**Solutions**:
```javascript
// Implement heartbeat (shown in best practices)
// Reduce message frequency
// Monitor server resources
```

#### Issue 3: "Invalid frame received"

**Causes**:
- Incorrect masking (client must mask)
- Invalid UTF-8 in text frames
- Corrupted data

**Solutions**:
```javascript
// Ensure proper encoding
const message = JSON.stringify(data);
ws.send(message); // Automatically handles encoding

// For binary, use proper types
ws.send(new Uint8Array([...]));
```

#### Issue 4: Memory leaks

**Causes**:
- Not cleaning up event listeners
- Holding references to closed connections
- Not clearing intervals

**Solutions**:
```javascript
// Proper cleanup
ws.addEventListener('close', () => {
    clearInterval(heartbeatInterval);
    ws = null;
});

// Server-side cleanup
ws.on('close', () => {
    // Remove from connection pool
    clients.delete(ws);
});
```

### Debugging Tools

#### Browser DevTools

```javascript
// Chrome/Firefox DevTools > Network > WS tab
// Shows:
// - Handshake details
// - All frames sent/received
// - Frame timing
// - Close codes
```

#### Wireshark

```
Filter: websocket
Shows raw frame data including:
- Masking keys
- Opcodes
- Payload (if not encrypted)
```

#### Command-line Testing

```bash
# Install wscat
npm install -g wscat

# Connect to WebSocket server
wscat -c wss://example.com/socket

# Send message
> Hello Server

# Receive message
< Hello Client
```

---

## Appendix: Quick Reference

### Frame Header Bytes

```
FIN RSV Opcode: Byte 0
MASK Payload:   Byte 1
```

### Common Opcodes

- `0x1`: Text
- `0x2`: Binary
- `0x8`: Close
- `0x9`: Ping
- `0xA`: Pong

### Close Codes

- `1000`: Normal
- `1001`: Going Away
- `1006`: Abnormal (no close frame)

### WebSocket States

```javascript
WebSocket.CONNECTING (0)
WebSocket.OPEN (1)
WebSocket.CLOSING (2)
WebSocket.CLOSED (3)
```

---

**End of Documentation**

For official specification, see **RFC 6455**: https://tools.ietf.org/html/rfc6455