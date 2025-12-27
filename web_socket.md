# WebSocket: A Systems-Level Dissection

## Part 1: The Fundamental Problem Space

### Why WebSocket Exists: The TCP Half-Duplex Trap in HTTP

HTTP is built on TCP, which is inherently **full-duplex** at the transport layer. Yet HTTP's request-response model creates an artificial **half-duplex constraint** at the application layer. The client must initiate every exchange; the server cannot push data without being asked.

Before WebSocket, we had grotesque workarounds:

**Long Polling**: Client sends HTTP request → Server holds connection open → Eventually responds → Client immediately reconnects. This creates:
- Connection overhead: TCP handshake + TLS handshake for every "push"
- Header bloat: 500-1000 bytes of HTTP headers per message
- Resource exhaustion: Server maintains connection state across reconnections

**Server-Sent Events (SSE)**: Better, but still unidirectional. Server pushes over a persistent HTTP connection, but client must use separate HTTP requests for upstream data.

WebSocket tears down this artificial barrier and exposes the underlying TCP full-duplex nature to the application layer.

---

## Part 2: Protocol Architecture — From TCP to Application

### The Connection Lifecycle

```
┌─────────────────────────────────────────────────────────┐
│  APPLICATION LAYER (JavaScript/C/Go/Rust)               │
│  ws.send(), ws.onmessage                                │
└────────────────────┬────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────┐
│  WEBSOCKET PROTOCOL LAYER (RFC 6455)                    │
│  • Framing (opcode, masking, fragmentation)             │
│  • Control frames (ping/pong/close)                     │
│  • UTF-8 validation (text frames)                       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────┐
│  TCP LAYER (Kernel Space)                               │
│  • Socket buffers (sk_buff)                             │
│  • Flow control (sliding window)                        │
│  • Congestion control (CUBIC/BBR)                       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────┐
│  IP LAYER → LINK LAYER → PHYSICAL                       │
└─────────────────────────────────────────────────────────┘
```

---

## Part 3: The Handshake — HTTP to WebSocket Upgrade

### Client-Side Initiation

When you call `new WebSocket("ws://example.com/chat")`, here's what happens under the hood:

```javascript
// Browser's internal WebSocket implementation (conceptual)
class WebSocketImpl {
    constructor(url) {
        // 1. Parse URL and extract components
        const { protocol, hostname, port, path } = parseURL(url);
        
        // 2. Create TCP socket via browser's network stack
        // This ultimately calls socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
        this.tcpSocket = createTCPSocket(hostname, port || 80);
        
        // 3. Generate cryptographic key for handshake
        this.secWebSocketKey = generateBase64Key(16); // 16 random bytes
        
        // 4. Send HTTP Upgrade request
        this.sendHandshake();
    }
    
    sendHandshake() {
        const request = 
            `GET /chat HTTP/1.1\r\n` +
            `Host: example.com\r\n` +
            `Upgrade: websocket\r\n` +
            `Connection: Upgrade\r\n` +
            `Sec-WebSocket-Key: ${this.secWebSocketKey}\r\n` +
            `Sec-WebSocket-Version: 13\r\n` +
            `Origin: http://example.com\r\n\r\n`;
        
        // Write to TCP socket
        this.tcpSocket.write(request);
    }
}
```

**At the Kernel Level:**

```c
// Simplified view of what happens in kernel space
// File: net/ipv4/tcp.c (Linux kernel)

int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size) {
    // 1. Check socket state
    if (sk->sk_state != TCP_ESTABLISHED)
        return -EPIPE;
    
    // 2. Allocate sk_buff (socket buffer)
    struct sk_buff *skb = alloc_skb(size + header_space, GFP_KERNEL);
    
    // 3. Copy data from userspace to kernel buffer
    copy_from_user(skb->data, msg->msg_iov[0].iov_base, size);
    
    // 4. Add to socket's send queue
    skb_queue_tail(&sk->sk_write_queue, skb);
    
    // 5. Trigger TCP send machinery
    tcp_push(sk, flags, mss_now, TCP_NAGLE_OFF);
    
    return size;
}
```

### Server-Side Handshake Processing

**Node.js ws Library (Simplified):**

```javascript
// File: lib/websocket-server.js (ws npm package)
const crypto = require('crypto');
const http = require('http');

class WebSocketServer {
    handleUpgrade(req, socket, head) {
        // 1. Validate Upgrade headers
        if (req.headers.upgrade?.toLowerCase() !== 'websocket') {
            socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
            return;
        }
        
        // 2. Extract Sec-WebSocket-Key
        const key = req.headers['sec-websocket-key'];
        if (!key) {
            socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
            return;
        }
        
        // 3. Compute acceptance hash (RFC 6455 magic)
        const GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
        const acceptKey = crypto
            .createHash('sha1')
            .update(key + GUID)
            .digest('base64');
        
        // 4. Send 101 Switching Protocols response
        const responseHeaders = [
            'HTTP/1.1 101 Switching Protocols',
            'Upgrade: websocket',
            'Connection: Upgrade',
            `Sec-WebSocket-Accept: ${acceptKey}`,
            '', ''
        ].join('\r\n');
        
        socket.write(responseHeaders);
        
        // 5. Socket is now a WebSocket — transition state
        this.completeUpgrade(socket, req, head);
    }
    
    completeUpgrade(socket, req, head) {
        const ws = new WebSocket(null);
        ws.setSocket(socket, head);
        
        // Socket now expects WebSocket frames, not HTTP
        socket.removeAllListeners('data');
        socket.on('data', (chunk) => this.handleFrame(ws, chunk));
        
        this.emit('connection', ws, req);
    }
}
```

### The Cryptographic Handshake — Why It Exists

The `Sec-WebSocket-Key` and `Sec-WebSocket-Accept` dance serves a specific purpose: **preventing cache poisoning attacks**.

```python
# The RFC 6455 algorithm
import hashlib
import base64

client_key = "dGhlIHNhbXBsZSBub25jZQ=="  # Client sends this
MAGIC_GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"  # RFC constant

# Server computes:
concatenated = client_key + MAGIC_GUID
sha1_hash = hashlib.sha1(concatenated.encode()).digest()
accept_key = base64.b64encode(sha1_hash).decode()

# Result: "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
```

**Why not just "Upgrade: websocket"?**

Older proxies might cache HTTP Upgrade responses. Without cryptographic validation, a MITM proxy could respond with a cached "101 Switching Protocols" to any client attempting a WebSocket connection, breaking the protocol. The key-accept mechanism ensures both endpoints genuinely understand WebSocket.

---

## Part 4: Frame Structure — The Binary Protocol

WebSocket is a **message-oriented protocol** on top of TCP's **byte-stream abstraction**. It defines frames to delimit messages.

### Base Frame Format (RFC 6455 Section 5.2)

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
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

### Bit-by-Bit Breakdown

**Byte 0 (First Byte):**
- **Bit 0 (FIN)**: 1 = final fragment, 0 = more fragments coming
- **Bits 1-3 (RSV1-3)**: Reserved for extensions (usually 0)
- **Bits 4-7 (Opcode)**: Frame type
  - `0x0`: Continuation frame
  - `0x1`: Text frame (UTF-8)
  - `0x2`: Binary frame
  - `0x8`: Connection close
  - `0x9`: Ping
  - `0xA`: Pong

**Byte 1 (Second Byte):**
- **Bit 0 (MASK)**: 1 if payload is masked (MUST be 1 for client→server)
- **Bits 1-7 (Payload Length)**: 
  - `0-125`: Actual length
  - `126`: Next 2 bytes contain 16-bit length
  - `127`: Next 8 bytes contain 64-bit length

**Masking Key (4 bytes, if MASK=1):**
Random 32-bit value used to XOR the payload.

**Payload:**
The actual message data.

---

## Part 5: Implementation Deep Dive

### Client-to-Server Frame: Encoding Process

```rust
// Conceptual Rust implementation
// Based on tokio-tungstenite and rust-websocket

use rand::Rng;

struct FrameHeader {
    fin: bool,
    opcode: OpCode,
    mask: bool,
    payload_len: u64,
    masking_key: Option<[u8; 4]>,
}

enum OpCode {
    Text = 0x1,
    Binary = 0x2,
    Close = 0x8,
    Ping = 0x9,
    Pong = 0xA,
}

impl WebSocket {
    fn send_text(&mut self, message: &str) -> Result<()> {
        let payload = message.as_bytes();
        let payload_len = payload.len() as u64;
        
        // Generate random masking key (client MUST mask)
        let masking_key: [u8; 4] = rand::thread_rng().gen();
        
        // Build frame header
        let mut frame = Vec::new();
        
        // Byte 0: FIN=1, RSV=0, Opcode=Text
        frame.push(0b10000001); // FIN | TEXT
        
        // Byte 1: MASK=1, Payload length
        if payload_len < 126 {
            frame.push(0x80 | payload_len as u8); // MASK | length
        } else if payload_len < 65536 {
            frame.push(0x80 | 126);
            frame.extend_from_slice(&(payload_len as u16).to_be_bytes());
        } else {
            frame.push(0x80 | 127);
            frame.extend_from_slice(&payload_len.to_be_bytes());
        }
        
        // Masking key
        frame.extend_from_slice(&masking_key);
        
        // Masked payload
        let masked_payload: Vec<u8> = payload
            .iter()
            .enumerate()
            .map(|(i, &byte)| byte ^ masking_key[i % 4])
            .collect();
        
        frame.extend_from_slice(&masked_payload);
        
        // Write to underlying TCP socket
        self.socket.write_all(&frame)?;
        self.socket.flush()?;
        
        Ok(())
    }
}
```

### Why Masking? The Proxy Poisoning Attack

Client frames MUST be masked to prevent **cache poisoning via proxy misinterpretation**.

Imagine a malicious client sends this unmasked payload to a proxy:

```
GET /admin HTTP/1.1\r\n
Host: evil.com\r\n\r\n
```

If the proxy doesn't understand WebSocket framing, it might interpret this as an HTTP request and forward it, potentially poisoning caches or bypassing security controls. Masking ensures that proxies can't accidentally interpret WebSocket data as HTTP.

**The XOR masking algorithm:**

```c
// mask[4] is the 4-byte masking key
// data is the payload
for (size_t i = 0; i < length; i++) {
    data[i] ^= mask[i % 4];
}
```

XOR properties:
- `A ^ B ^ B = A` (reversible with same key)
- Fast (single CPU instruction)
- Destroys HTTP-like patterns

---

## Part 6: Server-Side Frame Parsing

### State Machine Architecture

A robust WebSocket implementation uses a state machine to handle partial frames arriving across multiple TCP packets.

```go
// Simplified Go implementation inspired by gorilla/websocket
// File: conn.go

type connReader struct {
    conn   net.Conn
    state  readState
    buffer []byte
    
    // Partial frame state
    frameHeader   frameHeader
    bytesRead     int
    maskPos       int
}

type readState int

const (
    stateReadHeader readState = iota
    stateReadPayloadLength
    stateReadMaskingKey
    stateReadPayload
)

func (r *connReader) readFrame() (*Frame, error) {
    for {
        switch r.state {
        case stateReadHeader:
            // Read first 2 bytes
            buf := make([]byte, 2)
            _, err := io.ReadFull(r.conn, buf)
            if err != nil {
                return nil, err
            }
            
            r.frameHeader.fin = (buf[0] & 0x80) != 0
            r.frameHeader.opcode = OpCode(buf[0] & 0x0F)
            r.frameHeader.mask = (buf[1] & 0x80) != 0
            payloadLen := buf[1] & 0x7F
            
            // Validate: server MUST NOT accept unmasked frames from client
            if !r.frameHeader.mask {
                return nil, ErrUnmaskedFrame
            }
            
            if payloadLen < 126 {
                r.frameHeader.payloadLen = uint64(payloadLen)
                r.state = stateReadMaskingKey
            } else if payloadLen == 126 {
                r.state = stateReadPayloadLength
                r.bytesRead = 2 // Next 2 bytes
            } else {
                r.state = stateReadPayloadLength
                r.bytesRead = 8 // Next 8 bytes
            }
            
        case stateReadPayloadLength:
            buf := make([]byte, r.bytesRead)
            _, err := io.ReadFull(r.conn, buf)
            if err != nil {
                return nil, err
            }
            
            if r.bytesRead == 2 {
                r.frameHeader.payloadLen = uint64(binary.BigEndian.Uint16(buf))
            } else {
                r.frameHeader.payloadLen = binary.BigEndian.Uint64(buf)
            }
            
            r.state = stateReadMaskingKey
            
        case stateReadMaskingKey:
            _, err := io.ReadFull(r.conn, r.frameHeader.maskKey[:])
            if err != nil {
                return nil, err
            }
            r.state = stateReadPayload
            
        case stateReadPayload:
            payload := make([]byte, r.frameHeader.payloadLen)
            _, err := io.ReadFull(r.conn, payload)
            if err != nil {
                return nil, err
            }
            
            // Unmask payload
            for i := range payload {
                payload[i] ^= r.frameHeader.maskKey[i%4]
            }
            
            // Validate UTF-8 for text frames
            if r.frameHeader.opcode == OpCodeText {
                if !utf8.Valid(payload) {
                    return nil, ErrInvalidUTF8
                }
            }
            
            r.state = stateReadHeader // Reset for next frame
            
            return &Frame{
                Header:  r.frameHeader,
                Payload: payload,
            }, nil
        }
    }
}
```

### Handling TCP Packet Boundaries

**The Problem**: TCP is a byte stream. A WebSocket frame might arrive split across multiple `read()` syscalls.

```
Packet 1: [0x81 0x85 0x37 0xfa 0x21 0x3d]  // Header + mask
Packet 2: [0x7f 0x9f 0x4d 0x51 0x58]       // Payload
```

The parser must:
1. Buffer incomplete reads
2. Maintain state across syscalls
3. Avoid memory exhaustion from malicious fragmentation

**Linux Kernel Perspective:**

```c
// File: net/ipv4/tcp_input.c
// When data arrives on a TCP socket

void tcp_data_queue(struct sock *sk, struct sk_buff *skb) {
    // 1. Add skb to receive queue
    __skb_queue_tail(&sk->sk_receive_queue, skb);
    
    // 2. Update receive window
    tcp_rcv_space_adjust(sk);
    
    // 3. Signal userspace if data available
    sk->sk_data_ready(sk);
    
    // Userspace read() will drain sk_receive_queue
}
```

When your application calls `socket.read()`, it's ultimately calling:

```c
// File: net/socket.c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count) {
    struct file *file = fget_light(fd);
    struct socket *sock = file->private_data;
    
    // This copies from kernel sk_receive_queue to userspace buffer
    return sock->ops->recvmsg(sock, &msg, count, flags);
}
```

---

## Part 7: Control Frames — The Keep-Alive Machinery

### Ping/Pong: Heartbeat Protocol

WebSocket connections idle at the TCP layer appear alive (TCP doesn't timeout), but intermediaries (proxies, NATs) may close idle connections. Ping/Pong frames detect dead connections.

```python
# Python asyncio implementation
import asyncio
import time

class WebSocket:
    def __init__(self):
        self.last_pong = time.time()
        self.ping_interval = 30  # seconds
        self.ping_timeout = 10
        
    async def ping_loop(self):
        """Background task sending periodic pings"""
        while True:
            await asyncio.sleep(self.ping_interval)
            
            # Send ping frame (opcode 0x9)
            ping_frame = self.build_frame(
                opcode=0x9,
                payload=os.urandom(4)  # Random payload for tracking
            )
            await self.send_raw(ping_frame)
            
            # Wait for pong response
            try:
                await asyncio.wait_for(
                    self.pong_event.wait(),
                    timeout=self.ping_timeout
                )
                self.pong_event.clear()
            except asyncio.TimeoutError:
                # No pong received — connection dead
                await self.close(code=1006, reason="Ping timeout")
                break
    
    def on_frame_received(self, frame):
        if frame.opcode == 0x9:  # Ping
            # MUST respond with Pong containing same payload
            pong = self.build_frame(
                opcode=0xA,
                payload=frame.payload
            )
            self.send_raw(pong)
        
        elif frame.opcode == 0xA:  # Pong
            self.last_pong = time.time()
            self.pong_event.set()
```

**Chrome DevTools reveals this:**

```
WebSocket frame (PING): opcode=9, length=4
  Payload: 0x3a7f21c8
  
WebSocket frame (PONG): opcode=10, length=4
  Payload: 0x3a7f21c8  ← Same as PING
```

### Close Handshake: Graceful Termination

Unlike TCP's abrupt `FIN`, WebSocket has a two-way close handshake.

```
Client                          Server
   |                              |
   |---- Close Frame (code) ----->|
   |     (state: CLOSING)         |
   |                              |
   |<--- Close Frame (code) ------|
   |     (state: CLOSED)          |
   |                              |
   |---- TCP FIN ---------------->|
   |<--- TCP FIN -----------------|
```

**Close Frame Structure:**

```
+--------+--------+------------------+
| Opcode | Length | Status Code (2B) | Reason (UTF-8)
+--------+--------+------------------+
   0x8      n           1000            "Normal closure"
```

**Status Codes (RFC 6455 Section 7.4):**

```javascript
const CLOSE_CODES = {
    1000: "Normal Closure",
    1001: "Going Away",           // Server shutdown
    1002: "Protocol Error",       // Invalid frame
    1003: "Unsupported Data",     // Binary when expecting text
    1006: "Abnormal Closure",     // No close frame (TCP died)
    1007: "Invalid Frame Payload", // Invalid UTF-8
    1008: "Policy Violation",     // Generic server rejection
    1009: "Message Too Big",
    1011: "Internal Server Error",
    1015: "TLS Handshake Failed"
};
```

**Implementation:**

```c
// File: ws_close.c (simplified from libwebsockets)

int ws_close(struct websocket *ws, uint16_t code, const char *reason) {
    uint8_t close_frame[125 + 2];  // Max control frame payload
    size_t len = 2;  // Status code
    
    // Encode status code (big-endian)
    close_frame[0] = (code >> 8) & 0xFF;
    close_frame[1] = code & 0xFF;
    
    // Add optional reason (UTF-8, max 123 bytes)
    if (reason) {
        size_t reason_len = strlen(reason);
        if (reason_len > 123) reason_len = 123;
        memcpy(close_frame + 2, reason, reason_len);
        len += reason_len;
    }
    
    // Build frame header
    uint8_t header[2] = {
        0x88,  // FIN=1, Opcode=Close
        len    // Payload length (no masking for server)
    };
    
    // Send to socket
    write(ws->fd, header, 2);
    write(ws->fd, close_frame, len);
    
    // Wait for client's close frame (with timeout)
    ws->state = WS_CLOSING;
    
    // After receiving client close, close TCP socket
    close(ws->fd);
    ws->state = WS_CLOSED;
    
    return 0;
}
```

---

## Part 8: Fragmentation — Streaming Large Messages

WebSocket supports message fragmentation to stream data without buffering entire messages.

### Fragmentation Example

Sending a 10MB file without loading it all into memory:

```javascript
// Client-side implementation
class StreamingUpload {
    constructor(ws, file) {
        this.ws = ws;
        this.file = file;
        this.chunkSize = 64 * 1024; // 64KB chunks
    }
    
    async upload() {
        const reader = this.file.stream().getReader();
        let first = true;
        
        while (true) {
            const { done, value } = await reader.read();
            
            if (done) {
                // Send final fragment
                this.sendFragment(new Uint8Array(0), true, first);
                break;
            }
            
            // Send fragment
            this.sendFragment(value, false, first);
            first = false;
        }
    }
    
    sendFragment(chunk, fin, first) {
        const opcode = first ? 0x2 : 0x0; // Binary or Continuation
        const finBit = fin ? 0x80 : 0x00;
        
        // Build frame manually for demonstration
        const frame = new ArrayBuffer(2 + 4 + chunk.length);
        const view = new DataView(frame);
        
        view.setUint8(0, finBit | opcode);
        view.setUint8(1, 0x80 | chunk.length); // Masked
        
        // ... masking key and payload ...
        
        this.ws.send(frame); // Actually ws.send() handles this
    }
}
```

**Frame Sequence:**

```
Frame 1: FIN=0, Opcode=Binary, Payload=64KB
Frame 2: FIN=0, Opcode=Continuation, Payload=64KB
Frame 3: FIN=0, Opcode=Continuation, Payload=64KB
...
Frame N: FIN=1, Opcode=Continuation, Payload=32KB
```

**Server Reassembly:**

```go
type MessageBuffer struct {
    opcode  OpCode
    frames  [][]byte
    totalSize uint64
}

func (ws *WebSocket) readMessage() ([]byte, error) {
    var msgBuf MessageBuffer
    
    for {
        frame, err := ws.readFrame()
        if err != nil {
            return nil, err
        }
        
        // First frame: capture opcode
        if len(msgBuf.frames) == 0 {
            if frame.opcode == OpCodeContinuation {
                return nil, ErrUnexpectedContinuation
            }
            msgBuf.opcode = frame.opcode
        } else {
            // Subsequent frames must be continuations
            if frame.opcode != OpCodeContinuation {
                return nil, ErrFragmentationError
            }
        }
        
        msgBuf.frames = append(msgBuf.frames, frame.payload)
        msgBuf.totalSize += uint64(len(frame.payload))
        
        // Check size limit
        if msgBuf.totalSize > ws.maxMessageSize {
            ws.close(1009, "Message too large")
            return nil, ErrMessageTooLarge
        }
        
        // Final frame?
        if frame.fin {
            break
        }
    }
    
    // Concatenate all fragments
    result := make([]byte, msgBuf.totalSize)
    offset := 0
    for _, fragment := range msgBuf.frames {
        copy(result[offset:], fragment)
        offset += len(fragment)
    }
    
    return result, nil
}
```

---

## Part 9: Extensions — Compression and Beyond

### permessage-deflate: Transparent Compression

The `permessage-deflate` extension (RFC 7692) applies DEFLATE compression per-message.

**Handshake Negotiation:**

```http
Client → Server:
GET /chat HTTP/1.1
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits

Server → Client:
HTTP/1.1 101 Switching Protocols
Sec-WebSocket-Extensions: permessage-deflate; server_max_window_bits=15
```

**Frame Format Changes:**

```
Normal frame:
  [FIN=1, RSV1=0, Opcode=Text] [Length] [Payload]

Compressed frame:
  [FIN=1, RSV1=1, Opcode=Text] [Length] [Compressed Payload]
           ↑
     RSV1=1 indicates compression
```

**Implementation (C with zlib):**

```c
#include <zlib.h>

int compress_payload(const uint8_t *input, size_t input_len,
                    uint8_t **output, size_t *output_len) {
    z_stream strm;
    strm.zalloc = Z_NULL;
    strm.zfree = Z_NULL;
    strm.opaque = Z_NULL;
    
    // Initialize deflate with window bits = 15 (32KB window)
    if (deflateInit2(&strm, Z_DEFAULT_COMPRESSION, Z_DEFLATED,
                     -15, 8, Z_DEFAULT_STRATEGY) != Z_OK) {
        return -1;
    }
    
    // Allocate output buffer (worst case: input_len + 6)
    *output = malloc(input_len + 6);
    
    strm.avail_in = input_len;
    strm.next_in = (uint8_t *)input;
    strm.avail_out = input_len + 6;
    strm.next_out = *output;
    
    // Compress
    deflate(&strm, Z_SYNC_FLUSH);
    
    // RFC 7692: Remove trailing 0x00 0x00 0xff 0xff
    *output_len = strm.total_out - 4;
    
    deflateEnd(&strm);
    return 0;
}

int decompress_payload(const uint8_t *input, size_t input_len,
                       uint8_t **output, size_t *output_len) {
    z_stream strm = {0};
    
    // Initialize inflate
    if (inflateInit2(&strm, -15) != Z_OK) {
        return -1;
    }
    
    // Allocate output buffer (estimate 10x expansion)
    size_t out_capacity = input_len * 10;
    *output = malloc(out_capacity);
    
    strm.avail_in = input_len;
    strm.next_in = (uint8_t *)input;
    strm.avail_out = out_capacity;
    strm.next_out = *output;
    
    // RFC 7692: Append 0x00 0x00 0xff 0xff before inflating
    uint8_t *temp = malloc(input_len + 4);
    memcpy(temp, input, input_len);
    temp[input_len] = 0x00;
    temp[input_len + 1] = 0x00;
    temp[input_len + 2] = 0xff;
    temp[input_len + 3] = 0xff;
    
    strm.avail_in = input_len + 4;
    strm.next_in = temp;
    
    int ret = inflate(&strm, Z_SYNC_FLUSH);
    
    *output_len = strm.total_out;
    
    free(temp);
    inflateEnd(&strm);
    
    return (ret == Z_OK || ret == Z_STREAM_END) ? 0 : -1;
}
```

**Compression Efficiency:**

```json
// Uncompressed: 156 bytes
{"type":"chat","user":"alice","message":"Hello","timestamp":1640000000000}

// Compressed (permessage-deflate): 78 bytes (50% reduction)
// Actual bytes: 0x78 0x9c 0xab 0x56 0x2a 0xa9 0x2c 0x48 0x55 ...
```

For repetitive JSON/XML, compression ratios of 70-90% are common.

---

## Part 10: Security Considerations

### Origin-Based Security Model

Unlike TCP sockets, WebSocket connections from browsers enforce **same-origin policy via the Origin header**.

```javascript
// Browser automatically sends:
Origin: https://example.com

// Server MUST validate:
app.ws('/chat', (ws, req) => {
    const origin = req.headers.origin;
    const allowedOrigins = ['https://example.com', 'https://app.example.com'];
    
    if (!allowedOrigins.includes(origin)) {
        ws.close(1008, 'Origin not allowed');
        return;
    }
    
    // Connection allowed
});
```

**CSRF Attack Vector:**

Without Origin validation, an attacker's site `evil.com` could:

```html
<script>
const ws = new WebSocket('wss://bank.com/transfer');
ws.onopen = () => {
    ws.send(JSON.stringify({
        from: 'victim',
        to: 'attacker',
        amount: 10000
    }));
};
</script>
```

The browser will include the victim's cookies automatically. Origin header prevents this.

### TLS: wss:// Protocol

`wss://` is WebSocket over TLS, analogous to `https://`.

**Handshake Flow:**

```
1. TCP SYN/SYN-ACK/ACK
2. TLS ClientHello → ServerHello
3. TLS Certificate verification
4. TLS key exchange (ECDHE)
5. HTTP GET /chat HTTP/1.1 (over TLS)
6. HTTP 101 Switching Protocols (over TLS)
7. WebSocket frames (over TLS)
```

**OpenSSL Integration:**

```c
// File: wss_server.c
#include <openssl/ssl.h>
#include <openssl/err.h>

SSL_CTX *create_context() {
    SSL_CTX *ctx = SSL_CTX_new(TLS_server_method());
    
    // Load certificate
    SSL_CTX_use_certificate_file(ctx, "cert.pem", SSL_FILETYPE_PEM);
    SSL_CTX_use_PrivateKey_file(ctx, "key.pem", SSL_FILETYPE_PEM);
    
    // Enable TLS 1.3
    SSL_CTX_set_min_proto_version(ctx, TLS1_3_VERSION);
    
    return ctx;
}

void handle_client(int client_fd) {
    SSL *ssl = SSL_new(ctx);
    SSL_set_fd(ssl, client_fd);
    
    // TLS handshake
    if (SSL_accept(ssl) <= 0) {
        ERR_print_errors_fp(stderr);
        return;
    }
    
    // Read HTTP Upgrade request through TLS
    char buffer[4096];
    SSL_read(ssl, buffer, sizeof(buffer));
    
    // ... WebSocket handshake ...
    
    // All subsequent frames go through SSL_read/SSL_write
    while (1) {
        uint8_t header[2];
        SSL_read(ssl, header, 2);
        // ... parse frame ...
    }
}
```

---

## Part 11: Performance Characteristics

### Memory Usage: Per-Connection Overhead

**Typical WebSocket connection footprint:**

```
┌─────────────────────────────────────┐
│ TCP socket (kernel)        ~4-8 KB  │ ← sk_buff queues
│ TLS session (OpenSSL)      ~20 KB   │ ← Cipher state
│ WebSocket parser state     ~2 KB    │ ← Frame buffer
│ Application buffer         ~16 KB   │ ← Message queue
├─────────────────────────────────────┤
│ TOTAL per connection:      ~42 KB   │
└─────────────────────────────────────┘
```

**C10K Problem:**
- 10,000 connections × 42KB = 420MB minimum
- Plus application state (e.g., chat history)

**Modern servers handle C100K (100,000 connections):**

```go
// Go's goroutine-per-connection model
func main() {
    ln, _ := net.Listen("tcp", ":8080")
    
    for {
        conn, _ := ln.Accept()
        go handleWebSocket(conn) // Only ~2KB stack per goroutine
    }
}
```

### Latency Analysis

**HTTP Long Polling vs WebSocket:**

```
Long Polling (per message):
  ┌─ Client: HTTP request   ────────────────┐
  │  - TCP handshake (if new conn): 1 RTT   │
  │  - TLS handshake (if new conn): 2 RTT   │
  │  - HTTP request: 1 RTT                  │
  │  - HTTP response: 1 RTT                 │
  └─ Total: 2-5 RTT ────────────────────────┘

WebSocket (per message):
  ┌─ Client: WS frame ──────┐
  │  - Send frame: 1 RTT    │
  └─ Total: 1 RTT ──────────┘
  
  Handshake overhead amortized over connection lifetime
```

**Practical Measurement:**

```javascript
// Client-side latency test
const ws = new WebSocket('wss://example.com');
const measurements = [];

ws.onopen = () => {
    setInterval(() => {
        const start = performance.now();
        const pingId = Math.random();
        
        ws.send(JSON.stringify({ type: 'ping', id: pingId }));
        
        // Measure round-trip time
        const originalOnMessage = ws.onmessage;
        ws.onmessage = (event) => {
            const msg = JSON.parse(event.data);
            if (msg.type === 'pong' && msg.id === pingId) {
                const rtt = performance.now() - start;
                measurements.push(rtt);
                console.log(`RTT: ${rtt.toFixed(2)}ms`);
            }
            originalOnMessage?.(event);
        };
    }, 1000);
};
```

Typical results:
- WebSocket RTT: 15-50ms (1 RTT)
- HTTP long poll: 50-200ms (2-4 RTT including reconnect)

---

## Part 12: Real-World Implementation Examples

### Browser Native Implementation (Chromium)

**File: `third_party/blink/renderer/modules/websockets/websocket_channel_impl.cc`**

```cpp
// Simplified from Chromium source
class WebSocketChannelImpl : public WebSocketChannel {
 public:
  void SendFrame(bool fin, WebSocketOpCode opcode,
                 const char* data, size_t size) {
    // 1. Build frame header
    std::vector<char> frame;
    frame.push_back((fin ? 0x80 : 0x00) | opcode);
    
    // 2. Encode length
    if (size < 126) {
      frame.push_back(0x80 | size); // Masked
    } else if (size < 65536) {
      frame.push_back(0x80 | 126);
      frame.push_back((size >> 8) & 0xFF);
      frame.push_back(size & 0xFF);
    } else {
      frame.push_back(0x80 | 127);
      for (int i = 7; i >= 0; --i) {
        frame.push_back((size >> (i * 8)) & 0xFF);
      }
    }
    
    // 3. Generate masking key
    uint32_t masking_key = base::RandUint64();
    for (int i = 0; i < 4; ++i) {
      frame.push_back((masking_key >> (i * 8)) & 0xFF);
    }
    
    // 4. Mask and append payload
    for (size_t i = 0; i < size; ++i) {
      frame.push_back(data[i] ^ ((masking_key >> ((i % 4) * 8)) & 0xFF));
    }
    
    // 5. Send to network stack
    socket_->Write(frame.data(), frame.size(),
                   base::BindOnce(&WebSocketChannelImpl::OnWriteComplete,
                                  weak_ptr_factory_.GetWeakPtr()));
  }
  
 private:
  std::unique_ptr<StreamSocket> socket_;
  WebSocketInflateStream inflate_stream_;
  WebSocketDeflateStream deflate_stream_;
};
```

### Node.js ws Library Architecture

**File: `node_modules/ws/lib/receiver.js`**

```javascript
class Receiver extends Writable {
  constructor(extensions, maxPayload) {
    super();
    
    this._state = GET_INFO;
    this._bufferedBytes = 0;
    this._fragments = [];
    this._compressed = false;
    
    // Extension handlers
    this._extensions = extensions;
    this._maxPayload = maxPayload;
    
    // Partial frame state
    this._info = null;
    this._payloadLength = 0;
    this._mask = null;
  }
  
  _write(chunk, encoding, callback) {
    if (this._bufferedBytes + chunk.length > this._maxPayload) {
      this._error(1009, 'Max payload size exceeded');
      return;
    }
    
    this._bufferedBytes += chunk.length;
    this._buffers.push(chunk);
    
    this.startLoop(callback);
  }
  
  startLoop(callback) {
    while (true) {
      switch (this._state) {
        case GET_INFO:
          if (this._bufferedBytes < 2) {
            callback();
            return;
          }
          
          this.getInfo();
          break;
          
        case GET_PAYLOAD_LENGTH_16:
          if (this._bufferedBytes < 2) {
            callback();
            return;
          }
          
          this._payloadLength = this.readUInt16BE();
          this._state = this._masked ? GET_MASK : GET_DATA;
          break;
          
        case GET_MASK:
          if (this._bufferedBytes < 4) {
            callback();
            return;
          }
          
          this._mask = this.consume(4);
          this._state = GET_DATA;
          break;
          
        case GET_DATA:
          if (this._bufferedBytes < this._payloadLength) {
            callback();
            return;
          }
          
          const data = this.consume(this._payloadLength);
          
          // Unmask if needed
          if (this._masked) {
            unmask(data, this._mask);
          }
          
          // Decompress if extension active
          if (this._compressed) {
            this._inflate.decompress(data, this._fin, (err, result) => {
              if (err) return this._error(1007, 'Invalid compressed data');
              this.processData(result);
            });
          } else {
            this.processData(data);
          }
          
          this._state = GET_INFO;
          break;
      }
    }
  }
}
```

---

## Part 13: Comparison with Alternatives

### WebSocket vs Server-Sent Events (SSE)

| Aspect | WebSocket | SSE |
|--------|-----------|-----|
| **Directionality** | Full-duplex | Server → Client only |
| **Transport** | TCP (custom protocol) | HTTP (text/event-stream) |
| **Framing** | Binary frames | Text lines with `data:` prefix |
| **Browser Support** | Universal | Universal (except IE) |
| **Reconnection** | Manual | Automatic (EventSource API) |
| **Message Format** | Binary or text | Text only |
| **Firewall/Proxy** | Often blocked on non-80/443 | Works everywhere (HTTP) |

**SSE Frame Format:**

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

data: {"message": "Hello"}

data: {"message": "World"}

event: customEvent
data: {"value": 42}
id: 123

```

**When to use SSE:**
- Unidirectional server push (stock tickers, notifications)
- HTTP/2 environments (multiplexing mitigates overhead)
- Firewall-restricted networks

**When to use WebSocket:**
- Bidirectional communication (chat, gaming)
- Low-latency requirements (<10ms)
- Binary data (audio/video streaming)

### WebSocket vs HTTP/2 Server Push

HTTP/2 Server Push is NOT a WebSocket replacement:

```
HTTP/2 Server Push:
  - Server proactively sends responses for anticipated requests
  - Still request-response paradigm
  - Example: Push CSS/JS with HTML response

WebSocket:
  - Server sends arbitrary data at any time
  - No request-response coupling
  - Example: Real-time stock price updates
```

### WebSocket vs gRPC Streaming

gRPC uses HTTP/2 streaming, which differs from WebSocket:

```protobuf
// gRPC bidirectional stream
service Chat {
  rpc Stream(stream Message) returns (stream Message);
}
```

**Under the hood:**

```
gRPC:
  HTTP/2 stream (single TCP connection, multiplexed)
  ├─ HEADERS frame (RPC metadata)
  ├─ DATA frames (Protobuf messages)
  └─ TRAILERS frame (status, errors)

WebSocket:
  Single TCP connection, custom framing
  ├─ No HTTP/2 multiplexing overhead
  └─ Raw frames (minimal header)
```

**Performance:**
- WebSocket: ~2-byte overhead per frame
- gRPC: ~9-byte HTTP/2 frame header + Protobuf overhead

**When to use gRPC:**
- Microservices with strongly-typed contracts
- Need multiplexing (many streams over one connection)
- RPC semantics (request/response pairing)

---

## Part 14: Debugging and Introspection

### Chrome DevTools Network Tab

```
WebSocket Frame Viewer:
┌────────────────────────────────────────┐
│ ↑ Data (text)  | Length: 23 | Fin      │
│ {"type":"ping","ts":123456}            │
├────────────────────────────────────────┤
│ ↓ Data (text)  | Length: 23 | Fin      │
│ {"type":"pong","ts":123457}            │
└────────────────────────────────────────┘

Frame Details:
  Opcode: Text (1)
  Mask: Yes
  Payload Length: 23
  Masking Key: 0x3a7f21c8
```

### Wireshark Dissection

```
Frame 42: 156 bytes on wire
  Ethernet II
  Internet Protocol Version 4
  Transmission Control Protocol
    Source Port: 443
    Destination Port: 54321
    [Sequence number: 12345]
  WebSocket
    1... .... = Fin: True
    .000 .... = Reserved: 0x0
    .... 0001 = Opcode: Text (1)
    1... .... = Mask: True
    .001 0111 = Payload length: 23
    Masking-Key: 3a7f21c8
    Payload: 7b 22 74 79 70 65 22 3a 22 70 69 6e 67 22 ...
    [Unmasked Payload: {"type":"ping","ts":123456}]
```

### tcpdump Command

```bash
# Capture WebSocket traffic
tcpdump -i any -s0 -w ws.pcap 'tcp port 443 or tcp port 80'

# Filter WebSocket upgrade
tcpdump -i any -A 'tcp and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' | grep -i "upgrade: websocket"

# Decrypt wss:// traffic (requires SSLKEYLOGFILE)
export SSLKEYLOGFILE=/tmp/sslkeys.log
wireshark -o "tls.keylog_file:${SSLKEYLOGFILE}" ws.pcap
```

---

## Part 15: Advanced Topics

### Subprotocols: Application-Level Protocols

WebSocket itself is a transport. Subprotocols define message semantics.

```javascript
// Client requests STOMP subprotocol
const ws = new WebSocket('ws://example.com', 'stomp');

// Handshake:
// Client → Server:
//   Sec-WebSocket-Protocol: stomp
//
// Server → Client:
//   Sec-WebSocket-Protocol: stomp
```

**STOMP (Simple Text Oriented Messaging Protocol):**

```
SEND
destination:/queue/trades
content-type:application/json

{"symbol":"AAPL","price":150.25}^@

```

**MQTT over WebSocket:**

```javascript
// MQTT connects over WebSocket
const client = mqtt.connect('wss://broker.example.com', {
    protocolId: 'MQTT',
    protocolVersion: 4
});

// MQTTcontrol packets are binary frames inside WebSocket
```

### Load Balancing WebSocket Connections

**The Sticky Session Problem:**

Traditional load balancers use round-robin:

```
Client A → LB → Server 1 (HTTP request)
Client A → LB → Server 2 (Next HTTP request)
```

But WebSocket connections are stateful:

```
Client A → LB → Server 1 (WebSocket upgrade)
          (connection persists on Server 1)

Client A sends message → LB → ???
  ↓
  Must route to Server 1 (sticky session)
```

**Solution 1: IP Hash**

```nginx
# nginx.conf
upstream websocket_backend {
    ip_hash;  # Same IP always goes to same server
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080;
}

server {
    listen 443 ssl;
    
    location /ws {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;  # 24 hours
    }
}
```

**Solution 2: Centralized State (Redis)**

```javascript
// Server-agnostic approach
const redis = require('redis');
const client = redis.createClient();

wss.on('connection', (ws, req) => {
    const userId = authenticateUser(req);
    
    // Store connection → server mapping
    client.set(`ws:${userId}`, process.env.SERVER_ID);
    
    ws.on('close', () => {
        client.del(`ws:${userId}`);
    });
});

// When sending message to user
async function sendToUser(userId, message) {
    const serverId = await client.get(`ws:${userId}`);
    
    if (serverId === process.env.SERVER_ID) {
        // Local connection
        const ws = localConnections.get(userId);
        ws.send(message);
    } else {
        // Remote connection — use pub/sub
        redis.publish(`server:${serverId}`, JSON.stringify({
            userId,
            message
        }));
    }
}
```

### Horizontal Scaling with Redis Pub/Sub

```javascript
// server.js (each instance runs this)
const Redis = require('ioredis');
const subscriber = new Redis();
const publisher = new Redis();

const localConnections = new Map(); // userId → WebSocket

// Subscribe to this server's channel
subscriber.subscribe(`server:${process.env.SERVER_ID}`);

subscriber.on('message', (channel, data) => {
    const { userId, message } = JSON.parse(data);
    const ws = localConnections.get(userId);
    if (ws) {
        ws.send(message);
    }
});

// When client connects
wss.on('connection', (ws, req) => {
    const userId = authenticateUser(req);
    localConnections.set(userId, ws);
    
    ws.on('message', (data) => {
        const msg = JSON.parse(data);
        
        // Broadcast to all users
        publisher.publish('broadcast', JSON.stringify({
            from: userId,
            data: msg
        }));
    });
});

// Handle broadcasts
subscriber.subscribe('broadcast');
subscriber.on('message', (channel, data) => {
    if (channel === 'broadcast') {
        const { from, data: msg } = JSON.parse(data);
        
        // Send to all local connections except sender
        for (const [userId, ws] of localConnections) {
            if (userId !== from) {
                ws.send(JSON.stringify(msg));
            }
        }
    }
});
```

**Architecture:**

```
                    ┌─────────┐
                    │  Redis  │
                    │ Pub/Sub │
                    └────┬────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
    │Server 1 │     │Server 2 │     │Server 3 │
    │ 10K WS  │     │ 10K WS  │     │ 10K WS  │
    └────┬────┘     └────┬────┘     └────┬────┘
         │               │               │
    ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
    │Clients  │     │Clients  │     │Clients  │
    └─────────┘     └─────────┘     └─────────┘
```

---

## Part 16: Edge Cases and Gotchas

### The Half-Open Connection Problem

**Scenario:** Client crashes without sending close frame. Server believes connection is still open.

```python
# Without heartbeat
class WebSocketServer:
    def handle_connection(self, ws):
        while True:
            try:
                msg = ws.recv()  # Blocks forever if client died
                self.process(msg)
            except:
                break

# Server accumulates ghost connections
# Eventually: "Too many open files" error
```

**Solution: Aggressive Timeouts**

```python
import time

class WebSocketWithTimeout:
    def __init__(self, ws):
        self.ws = ws
        self.last_activity = time.time()
        self.timeout = 60  # seconds
        
    def recv_with_timeout(self):
        self.ws.settimeout(self.timeout)
        try:
            msg = self.ws.recv()
            self.last_activity = time.time()
            return msg
        except socket.timeout:
            # Check if we got ping/pong recently
            if time.time() - self.last_activity > self.timeout:
                raise ConnectionTimeout()
```

### The Buffering Backpressure Problem

**Scenario:** Server sends faster than client can consume. TCP send buffer fills up.

```javascript
// Server sends 1000 messages/sec
setInterval(() => {
    for (let i = 0; i < 1000; i++) {
        ws.send(generateLargeMessage());  // Each 10KB
    }
}, 1000);

// Client on slow network (1 Mbps)
// Can only consume ~125 KB/sec
// But server sends 10 MB/sec
```

**What happens at TCP level:**

```c
// Kernel's TCP send buffer (net/ipv4/tcp_output.c)
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size) {
    // Check send buffer space
    if (sk->sk_wmem_queued + size > sk->sk_sndbuf) {
        // Buffer full!
        if (flags & MSG_DONTWAIT)
            return -EAGAIN;  // Non-blocking: return immediately
        else
            wait_for_buffer_space(sk);  // Blocking: sleep until space
    }
    
    // Copy to send buffer
    skb = tcp_write_queue_tail(sk);
    // ...
}
```

**Solution: Monitor bufferedAmount**

```javascript
// Browser WebSocket
function sendWithBackpressure(ws, data) {
    if (ws.bufferedAmount > 1024 * 1024) {  // 1MB threshold
        // Too much queued data
        console.warn('Backpressure detected, slowing down');
        return false;
    }
    
    ws.send(data);
    return true;
}

// Node.js ws library
function sendWithBackpressure(ws, data) {
    const canSend = ws.bufferedAmount < 1024 * 1024;
    
    if (canSend) {
        ws.send(data, (err) => {
            if (err) console.error('Send failed:', err);
        });
    }
    
    return canSend;
}
```

### The UTF-8 Fragmentation Trap

**Problem:** Text frames MUST be valid UTF-8. But what if a multi-byte character is split across fragments?

```
UTF-8 character: € (U+20AC)
Bytes: E2 82 AC

Fragment 1: FIN=0, Payload=[... E2 82]  ← Incomplete UTF-8
Fragment 2: FIN=1, Payload=[AC ...]      ← Completes character
```

**RFC 6455 specifies:**
- Individual fragments need not be valid UTF-8
- Only the **concatenated** message must be valid UTF-8

**Implementation:**

```go
type TextFrameValidator struct {
    incompleteUTF8 []byte  // Carries over incomplete bytes
}

func (v *TextFrameValidator) Validate(fragment []byte, fin bool) error {
    // Prepend incomplete bytes from previous fragment
    data := append(v.incompleteUTF8, fragment...)
    
    if fin {
        // Final fragment: entire message must be valid
        if !utf8.Valid(data) {
            return ErrInvalidUTF8
        }
        v.incompleteUTF8 = nil
    } else {
        // Non-final fragment: may end mid-character
        // Find last complete rune
        for i := len(data) - 1; i >= 0; i-- {
            r, size := utf8.DecodeRune(data[i:])
            if r != utf8.RuneError {
                // Found complete rune
                v.incompleteUTF8 = data[i+size:]
                return nil
            }
            
            // Check if we're at a UTF-8 start byte
            if data[i]&0xC0 == 0xC0 {
                // This is a start byte, save everything from here
                v.incompleteUTF8 = data[i:]
                return nil
            }
        }
    }
    
    return nil
}
```

### The Proxy Timeout Trap

**Scenario:** Cloud load balancers (AWS ALB, GCP LB) have idle timeouts (typically 60s).

```
Client ←→ Load Balancer ←→ Server
          (60s timeout)

[40 seconds of silence]
[59 seconds of silence]
→ Load Balancer closes connection
→ Client doesn't know (TCP doesn't notify)
→ Client sends message into void
```

**Solution: Application-Level Ping/Pong**

```javascript
// Client-side keep-alive
class WebSocketWithKeepalive {
    constructor(url) {
        this.ws = new WebSocket(url);
        this.pingInterval = 30000;  // 30 seconds (< 60s timeout)
        
        this.ws.onopen = () => {
            this.startPingLoop();
        };
        
        this.ws.onclose = () => {
            clearInterval(this.pingTimer);
        };
    }
    
    startPingLoop() {
        this.pingTimer = setInterval(() => {
            if (this.ws.readyState === WebSocket.OPEN) {
                this.ws.send(JSON.stringify({ type: 'ping' }));
            }
        }, this.pingInterval);
    }
}
```

---

## Part 17: Language-Specific Implementations

### Python: websockets Library (asyncio)

```python
# server.py
import asyncio
import websockets
import json

connected_clients = set()

async def handler(websocket, path):
    # Register client
    connected_clients.add(websocket)
    
    try:
        async for message in websocket:
            # Parse message
            data = json.loads(message)
            
            # Broadcast to all clients
            await asyncio.gather(
                *[client.send(message) for client in connected_clients]
            )
    
    finally:
        # Unregister client
        connected_clients.remove(websocket)

# Start server with asyncio
start_server = websockets.serve(handler, "localhost", 8765)
asyncio.get_event_loop().run_until_complete(start_server)
asyncio.get_event_loop().run_forever()
```

**Under the hood** (from `websockets` library source):

```python
# File: websockets/protocol.py
class WebSocketCommonProtocol:
    async def recv(self):
        """Receive the next message."""
        # Reassemble fragmented message
        frame = await self.read_data_frame(max_size=self.max_size)
        
        if frame.opcode == OP_TEXT:
            text = frame.data.decode('utf-8')
            return text
        elif frame.opcode == OP_BINARY:
            return frame.data
    
    async def read_data_frame(self, max_size):
        """Read a complete data frame (handling fragmentation)."""
        frame = await self.read_frame(max_size)
        
        # Handle control frames immediately
        if frame.opcode in (OP_CLOSE, OP_PING, OP_PONG):
            await self.handle_control_frame(frame)
            return await self.read_data_frame(max_size)
        
        # First fragment or unfragmented message
        if frame.opcode in (OP_TEXT, OP_BINARY):
            self.opcode = frame.opcode
            self.frames = [frame]
        
        # Read continuation frames
        while not frame.fin:
            frame = await self.read_frame(max_size)
            self.frames.append(frame)
        
        # Concatenate all fragments
        data = b''.join(f.data for f in self.frames)
        return Frame(self.opcode, data, fin=True)
```

### Rust: tokio-tungstenite (Zero-Copy Design)

```rust
// server.rs
use tokio::net::{TcpListener, TcpStream};
use tokio_tungstenite::{accept_async, tungstenite::Message};
use futures_util::{StreamExt, SinkExt};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
    
    while let Ok((stream, _)) = listener.accept().await {
        tokio::spawn(handle_connection(stream));
    }
}

async fn handle_connection(stream: TcpStream) {
    let ws_stream = accept_async(stream).await.unwrap();
    let (mut write, mut read) = ws_stream.split();
    
    while let Some(msg) = read.next().await {
        match msg {
            Ok(Message::Text(text)) => {
                println!("Received: {}", text);
                
                // Echo back
                write.send(Message::Text(text)).await.unwrap();
            }
            
            Ok(Message::Binary(data)) => {
                // Zero-copy: data is Bytes (reference-counted)
                write.send(Message::Binary(data)).await.unwrap();
            }
            
            Ok(Message::Close(_)) => break,
            
            _ => {}
        }
    }
}
```

**Key optimization: Bytes type (zero-copy)**

```rust
// From bytes crate
pub struct Bytes {
    ptr: *const u8,
    len: usize,
    data: Arc<Vec<u8>>,  // Reference-counted
}

// Multiple Bytes can reference same underlying buffer
let original = Bytes::from(vec![1, 2, 3, 4, 5]);
let slice1 = original.slice(0..3);  // [1, 2, 3] — no copy!
let slice2 = original.slice(3..5);  // [4, 5] — no copy!
```

### Go: gorilla/websocket (Concurrency Model)

```go
// server.go
package main

import (
    "github.com/gorilla/websocket"
    "net/http"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
}

type Client struct {
    conn *websocket.Conn
    send chan []byte
}

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
}

func (h *Hub) run() {
    for {
        select {
        case client := <-h.register:
            h.clients[client] = true
            
        case client := <-h.unregister:
            delete(h.clients, client)
            close(client.send)
            
        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    // Client's send buffer full — drop client
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}

func (c *Client) readPump() {
    defer func() {
        hub.unregister <- c
        c.conn.Close()
    }()
    
    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            break
        }
        
        hub.broadcast <- message
    }
}

func (c *Client) writePump() {
    defer c.conn.Close()
    
    for message := range c.send {
        if err := c.conn.WriteMessage(websocket.TextMessage, message); err != nil {
            break
        }
    }
}

func serveWs(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        return
    }
    
    client := &Client{
        conn: conn,
        send: make(chan []byte, 256),
    }
    
    hub.register <- client
    
    // Spawn two goroutines per connection
    go client.writePump()  // Dedicated writer
    go client.readPump()   // Dedicated reader
}
```

**Go's Goroutine Advantage:**

```
Traditional threading:
  1 connection = 1 OS thread = 1-2 MB stack
  10,000 connections = 10-20 GB memory

Go goroutines:
  1 connection = 2 goroutines = ~8 KB total stack
  10,000 connections = ~80 MB memory
```

---

## Part 18: WebSocket over HTTP/3 and QUIC

### The Next Evolution: WebSocket over QUIC

HTTP/3 runs on QUIC (UDP-based), raising the question: **Can WebSocket run over HTTP/3?**

**Current Status (RFC 9220):** WebSocket over HTTP/3 is defined but not widely implemented.

```
Traditional WebSocket:
  TCP → TLS → HTTP/1.1 Upgrade → WebSocket frames

WebSocket over HTTP/3:
  UDP → QUIC (includes TLS 1.3) → HTTP/3 → WebSocket frames
```

**QUIC Advantages:**

1. **0-RTT Connection:** Resume previous connection without handshake
2. **No Head-of-Line Blocking:** Multiple streams don't block each other
3. **Connection Migration:** Survive IP address changes (WiFi → cellular)

**Frame Mapping:**

```
HTTP/3 uses QPACK for header compression:

CONNECT request:
  :method = CONNECT
  :protocol = websocket
  :scheme = https
  :path = /chat
  :authority = server.example.com

Response:
  :status = 200
  sec-websocket-protocol = chat

→ Now WebSocket frames flow over HTTP/3 STREAM
```

**Implementation Complexity:**

```rust
// Hypothetical implementation using quinn (QUIC library)
async fn websocket_over_quic(mut stream: SendStream) {
    // HTTP/3 CONNECT handshake
    send_connect_request(&mut stream).await;
    
    // Wait for 200 response
    let response = receive_response(&mut stream).await;
    
    // Now send WebSocket frames over QUIC stream
    let ws_frame = build_frame(OpCode::Text, "Hello");
    stream.write_all(&ws_frame).await;
    
    // QUIC handles:
    // - Congestion control
    // - Packet loss recovery
    // - Stream multiplexing
}
```

---

## Part 19: Performance Tuning Deep Dive

### Kernel Tuning for High-Concurrency WebSocket Servers

**Increase file descriptor limits:**

```bash
# /etc/sysctl.conf
fs.file-max = 2097152

# Per-process limit
ulimit -n 1048576

# /etc/security/limits.conf
* soft nofile 1048576
* hard nofile 1048576
```

**TCP Buffer Tuning:**

```bash
# Increase TCP buffer sizes
net.core.rmem_max = 134217728  # 128 MB
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# Enable TCP window scaling
net.ipv4.tcp_window_scaling = 1

# Increase backlog queue
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 8192
```

**Connection Tracking:**

```bash
# Increase conntrack table size
net.netfilter.nf_conntrack_max = 1048576
net.nf_conntrack_max = 1048576
```

### Application-Level Optimizations

**1. Message Batching:**

```javascript
// Bad: Send 1000 individual frames
for (let i = 0; i < 1000; i++) {
    ws.send(JSON.stringify({ type: 'update', id: i }));
}
// Result: 1000 WebSocket frames, 1000 syscalls

// Good: Batch into single message
const batch = [];
for (let i = 0; i < 1000; i++) {
    batch.push({ type: 'update', id: i });
}
ws.send(JSON.stringify(batch));
// Result: 1 WebSocket frame, 1 syscall
```

**2. Protocol Buffers over JSON:**

```protobuf
// message.proto
syntax = "proto3";

message ChatMessage {
  string user_id = 1;
  string text = 2;
  int64 timestamp = 3;
}
```

**Size Comparison:**

```json
// JSON: 78 bytes
{"user_id":"alice123","text":"Hello","timestamp":1640000000}

// Protobuf: 28 bytes (64% reduction)
\x0a\x08alice123\x12\x05Hello\x18\x80\x94\xeb\xdc\x83\x05
```

**3. Shared Memory for Broadcast (C/C++):**

```c
// For broadcasting to 10,000 connections:
// Instead of copying message 10,000 times, use shared memory

struct BroadcastMessage {
    atomic_int refcount;
    size_t length;
    char data[];
};

void broadcast(struct BroadcastMessage *msg) {
    atomic_fetch_add(&msg->refcount, num_clients);
    
    for (int i = 0; i < num_clients; i++) {
        // Each client gets pointer, not copy
        client_send_queue[i] = msg;
    }
}

void client_sent(struct BroadcastMessage *msg) {
    if (atomic_fetch_sub(&msg->refcount, 1) == 1) {
        // Last client sent — free message
        free(msg);
    }
}
```

---

## Part 20: Security Vulnerabilities and Mitigations

### 1. Denial of Service: Compression Bomb

**Attack:** Send small compressed payload that expands to gigabytes.

```
Compressed: 1 KB
Decompressed: 1 GB (compression ratio: 1000x)
```

**Mitigation:**

```javascript
const ws = new WebSocketServer({
    perMessageDeflate: {
        zlibDeflateOptions: {
            level: 6,  // Moderate compression
        },
        zlibInflateOptions: {
            chunkSize: 10 * 1024,  // 10KB chunks
        },
        threshold: 1024,  // Only compress > 1KB messages
        
        // CRITICAL: Limit decompressed size
        maxPayload: 10 * 1024 * 1024  // 10 MB max
    }
});
```

### 2. Slowloris-Style Attack

**Attack:** Send frames extremely slowly to exhaust server resources.

```python
import socket
import time

sock = socket.socket()
sock.connect(('target.com', 80))

# Send HTTP Upgrade byte-by-byte
request = "GET / HTTP/1.1\r\nUpgrade: websocket\r\n..."
for byte in request:
    sock.send(byte.encode())
    time.sleep(1)  # 1 second per byte
```

**Mitigation:**

```go
// Set aggressive read timeout during handshake
conn.SetReadDeadline(time.Now().Add(10 * time.Second))

// After handshake, use heartbeat timeout
go func() {
    ticker := time.NewTicker(30 * time.Second)
    for range ticker.C {
        if time.Since(client.lastActivity) > 60*time.Second {
            client.Close()
        }
    }
}()
```

### 3. Resource Exhaustion: Memory Fragmentation

**Attack:** Send many small messages to fragment server memory.

```javascript
// Attacker sends 1 million 10-byte messages
for (let i = 0; i < 1000000; i++) {
    ws.send('x'.repeat(10));
}

// Server allocates 1 million buffers
// Memory becomes highly fragmented
```

**Mitigation:**

```rust
// Use memory pool for common sizes
use slab::Slab;

struct MessagePool {
    small: Slab<Vec<u8>>,   // < 1KB
    medium: Slab<Vec<u8>>,  // 1KB - 64KB
    large: Slab<Vec<u8>>,   // > 64KB
}

impl MessagePool {
    fn alloc(&mut self, size: usize) -> Vec<u8> {
        match size {
            0..=1024 => self.small.vacant_entry().insert(Vec::with_capacity(1024)),
            1025..=65536 => self.medium.vacant_entry().insert(Vec::with_capacity(65536)),
            _ => Vec::with_capacity(size),
        }
    }
}
```

---

## Conclusion: The WebSocket Philosophy

WebSocket is fundamentally about **exposing TCP's full-duplex nature to applications** while maintaining **web security boundaries**.

Every design choice traces back to these goals:

- **Handshake via HTTP:** Reuse existing infrastructure (ports, proxies, TLS)
- **Masking:** Prevent cache poisoning without deep proxy awareness
- **Framing:** Restore message boundaries on byte-stream transport
- **Control frames:** Detect dead connections in NAT environments
- **Extensions:** Allow compression without protocol proliferation

It sits at a unique position in the stack:

```
Application:  Real-time semantics (chat, gaming, trading)
WebSocket:    Message framing, flow control, security
TLS:          Encryption, authentication
TCP:          Reliable, ordered byte stream
IP:           Best-effort packet delivery
```

When you call `ws.send()`, you're triggering a cascade through this entire stack — from JavaScript VM → Browser WebSocket implementation → TLS encryption → TCP windowing → IP routing → physical bits on wire.

Understanding this flow transforms WebSocket from a "magic real-time API" into a precisely engineered protocol that elegantly solves the HTTP request-response constraint while respecting the web's security model.