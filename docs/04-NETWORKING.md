# Project ECHO: Networking Architecture

## Table of Contents

1. [Transport Layer](#transport-layer)
2. [Protocol Stack](#protocol-stack)
3. [Binary Protocol](#binary-protocol)
4. [Serialization](#serialization)
5. [Compression](#compression)
6. [Connection Management](#connection-management)
7. [P2P Networking](#p2p-networking)
8. [Latency Optimization](#latency-optimization)

---

## Transport Layer

### QUIC/HTTP3 Primary Transport

**QUIC (Quick UDP Internet Connection):**
- Multiplexed streams over single UDP connection
- 0-RTT connection resumption (fast reconnects)
- Forward error correction (FEC) for packet loss
- Congestion control (CUBIC algorithm)
- Built-in TLS 1.3 encryption

```cpp
class QuicTransport {
    quiche::Connection* quic_conn;
    
    // Initialize connection
    void connect(const InetAddress& server) {
        config = quiche::Config::new(quiche::PROTOCOL_VERSION);
        config->set_application_protos(b"echo/1");
        config->set_max_idle_timeout(30000);  // 30 seconds
        config->set_initial_max_streams_bidi(1000);
        config->set_initial_max_streams_uni(1000);
        config->set_initial_max_data(10_MB);
        
        quic_conn = quiche::connect(nullptr, server, config);
    }
    
    // Send data on stream
    int send_on_stream(uint64_t stream_id, const Bytes& data) {
        return quic_conn->stream_send(stream_id, data.data(), data.size(), false);
    }
    
    // Receive data
    void process_packets(const std::vector<Bytes>& packets) {
        for (const auto& packet : packets) {
            quic_conn->recv(packet.data(), packet.size());
        }
    }
};
```

### Fallback: Custom UDP Protocol

For environments where QUIC is blocked:

```cpp
class CustomUdpTransport {
    std::vector<UdpSocket> sockets;  // Multiple sockets for parallelism
    
    struct UdpHeader {
        uint16_t packet_id;      // Sequence for ordering
        uint16_t total_packets;  // For fragmentation
        uint16_t packet_index;   // Which fragment is this
        uint8_t flags;           // ACK_REQUESTED, RELIABLE, etc.
        uint32_t timestamp;      // For RTT estimation
    };
    
    void send_reliable(const InetAddress& dest, const Bytes& data) {
        if (data.size() <= MAX_UDP_PAYLOAD) {
            // Single packet
            send_single_packet(dest, data);
        } else {
            // Fragment and send
            uint16_t total = (data.size() + MAX_PAYLOAD - 1) / MAX_PAYLOAD;
            for (uint16_t i = 0; i < total; ++i) {
                auto fragment = data.subrange(i * MAX_PAYLOAD, MAX_PAYLOAD);
                send_fragment(dest, i, total, fragment);
            }
        }
    }
};
```

---

## Protocol Stack

### Layered Architecture

```
┌─────────────────────────────────────────┐
│  Application Layer Messages             │
│  (EntityUpdate, Transaction, etc.)      │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Custom Binary Protocol Frame           │
│  (MessageType, Routing, Ordering)       │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  FlatBuffers Serialization              │
│  (Zero-copy, efficient)                 │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  Compression (Zstd/LZ4)                 │
│  (20-50% size reduction)                │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  TLS 1.3 Encryption (QUIC)              │
│  (AES-256-GCM or ChaCha20-Poly1305)    │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│  QUIC Transport (HTTP/3)                │
│  (Multiplexed UDP streams)              │
└─────────────────────────────────────────┘
```

---

## Binary Protocol

### Message Frame Format

```cpp
struct MessageFrame {
    // Header (fixed 32 bytes)
    struct {
        uint32_t frame_size;           // Total frame size (for framing)
        uint16_t message_type;         // MessageType enum
        uint16_t flags;                // COMPRESSED, REQUIRES_ACK, etc.
        uint32_t sequence_number;      // For ordering
        uint64_t timestamp;            // Sender's clock
        UUID sender_id;                // Who sent this (16 bytes)
        uint16_t priority;             // 0-255 (higher = more important)
        uint16_t reserved;             // For future use
    } header;
    
    // Payload
    uint32_t payload_size;             // Uncompressed size
    Bytes compressed_payload;          // FlatBuffers + compression
};
```

### Message Types

```cpp
enum class MessageType : uint16_t {
    // === Connection Management (0-99) ===
    Ping = 0,                     // Latency probe
    Pong = 1,                     // Pong response
    Hello = 2,                    // Initial handshake
    Goodbye = 3,                  // Clean disconnect
    
    // === State Synchronization (100-199) ===
    EntitySpawn = 100,            // New entity in world
    EntityUpdate = 101,           // Entity state changed
    EntityDespawn = 102,          // Entity removed
    RegionSync = 103,             // Sync entire region
    StateMerkleProof = 104,       // Prove state correctness
    
    // === Transactions (200-299) ===
    SubmitTransaction = 200,      // Player submits tx
    TransactionReceipt = 201,     // Server confirms receipt
    TransactionFinalized = 202,   // Tx is permanent
    
    // === Consensus (300-399) ===
    ProposeBlock = 300,           // Validator proposes block
    VoteBlock = 301,              // Validator votes yes/no
    FinalizeBlock = 302,          // Block is canonical
    SyncBlocks = 303,             // Request block range
    
    // === Chat & Social (400-499) ===
    ChatMessage = 400,            // Player sends message
    EmoteAnimation = 401,         // Play emote
    VoicePacket = 402,            // Voice audio frame
    
    // === Distributed Computing (500-599) ===
    ComputeTask = 500,            // Assign work
    ComputeResult = 501,          // Return result
    
    // === Admin (600-699) ===
    AdminCommand = 600,           // Server admin only
    MetricsReport = 601,          // Performance stats
};
```

### Message Routing

```cpp
class MessageRouter {
    std::unordered_map<uint16_t, Handler> handlers;
    
    using Handler = std::function<void(const MessageFrame&)>;
    
    void register_handler(MessageType type, Handler handler) {
        handlers[static_cast<uint16_t>(type)] = handler;
    }
    
    void route_message(const MessageFrame& frame) {
        auto it = handlers.find(frame.header.message_type);
        if (it != handlers.end()) {
            it->second(frame);
        } else {
            log_warning("Unknown message type: {}", frame.header.message_type);
        }
    }
};
```

---

## Serialization

### FlatBuffers Schema

Used for efficient zero-copy serialization:

```fbs
// echo.fbs - FlatBuffers schema

namespace Echo;

table Vec3 {
  x: float;
  y: float;
  z: float;
}

table Quaternion {
  x: float;
  y: float;
  z: float;
  w: float;
}

table EntityUpdatePayload {
  entity_id: string (required);
  position: Vec3;
  rotation: Quaternion;
  velocity: Vec3;
  animation_state: string;
  custom_data: [ubyte];
}

table TransactionPayload {
  actor_id: string (required);
  tx_type: uint16;
  timestamp: uint64;
  nonce: uint64;
  data: [ubyte];
}

root_type EntityUpdatePayload;
```

### Serialization/Deserialization

```cpp
template<typename T>
Bytes serialize_message(const T& msg) {
    FlatBufferBuilder builder;
    
    // Build message into buffer
    auto built = msg.build(builder);
    builder.Finish(built);
    
    // Get raw bytes
    auto data = builder.GetBufferPointer();
    auto size = builder.GetSize();
    
    return Bytes(data, data + size);
}

template<typename T>
Result<T> deserialize_message(const Bytes& data) {
    try {
        Verifier verifier(data.data(), data.size());
        if (!verifier.VerifyBuffer<T>(nullptr)) {
            return Error("Failed to verify buffer");
        }
        
        auto msg = GetRootAsEntityUpdatePayload(data.data());
        return T(*msg);
    } catch (const std::exception& e) {
        return Error(std::string("Deserialization failed: ") + e.what());
    }
}
```

---

## Compression

### Algorithm Selection

| Use Case | Algorithm | Ratio | Speed | Notes |
|----------|-----------|-------|-------|-------|
| State updates | Zstd level 3 | 30-40% | 200+ MB/s | Good compression |
| Real-time chat | LZ4 | 20-30% | 500+ MB/s | Ultra-fast |
| Blockchain data | Zstd level 11 | 50-60% | 20 MB/s | Can afford latency |

### Compression Implementation

```cpp
class CompressionCodec {
    enum class Algorithm { Zstd, Lz4 };
    
public:
    Bytes compress(const Bytes& data, Algorithm algo) {
        if (algo == Algorithm::Zstd) {
            return compress_zstd(data, 3);  // Level 3 for balance
        } else {
            return compress_lz4(data);
        }
    }
    
    Result<Bytes> decompress(const Bytes& data, Algorithm algo) {
        if (algo == Algorithm::Zstd) {
            return decompress_zstd(data);
        } else {
            return decompress_lz4(data);
        }
    }
    
private:
    Bytes compress_zstd(const Bytes& data, int level) {
        size_t bound = ZSTD_compressBound(data.size());
        Bytes compressed(bound);
        
        size_t result = ZSTD_compress(
            compressed.data(), bound,
            data.data(), data.size(),
            level
        );
        
        if (ZSTD_isError(result)) {
            return Bytes();  // Error
        }
        
        compressed.resize(result);
        return compressed;
    }
};
```

---

## Connection Management

### Connection Lifecycle

```cpp
enum class ConnectionState {
    Connecting,     // Handshake in progress
    Connected,      // Fully established
    Authenticating, // Waiting for auth
    Authenticated,  // Ready for game
    Idle,           // No recent activity
    Disconnecting,  // Graceful shutdown
    Disconnected,   // Closed
};

class NetworkSession {
    UUID session_id;
    ConnectionState state;
    InetAddress peer_address;
    quiche::Connection* quic_conn;
    
    uint64_t last_activity;
    uint64_t bytes_sent;
    uint64_t bytes_received;
    float rtt_ms;               // Round-trip time estimate
    
public:
    void on_packet_received(const Bytes& packet) {
        last_activity = now();
        bytes_received += packet.size();
        
        // Process packet
        quic_conn->recv(packet.data(), packet.size());
        
        // Extract and route messages
        while (true) {
            Bytes stream_data;
            uint64_t stream_id;
            
            if (!quic_conn->stream_recv(&stream_id, stream_data)) {
                break;
            }
            
            route_message(stream_id, stream_data);
        }
    }
    
    void send_message(const MessageFrame& frame) {
        Bytes serialized = serialize_frame(frame);
        Bytes compressed = compress(serialized);
        
        uint64_t stream_id = next_stream_id();
        quic_conn->stream_send(stream_id, compressed.data(), compressed.size(), true);
        
        bytes_sent += compressed.size();
    }
    
    void check_timeout() {
        uint64_t idle_time = now() - last_activity;
        if (idle_time > IDLE_TIMEOUT) {
            disconnect("Idle timeout");
        }
    }
};
```

---

## P2P Networking

### Mesh Network Topology

```
    Client A
       /  \
      /    \
  Relay1  Relay2
      \    /
       \  /
    Client B

// Direct connection when possible
// Relayed through server if NAT-blocked
```

### Peer Discovery

```cpp
class PeerDiscovery {
    std::vector<PeerInfo> known_peers;
    
    struct PeerInfo {
        UUID peer_id;
        InetAddress address;
        uint64_t last_seen;
        float reputation;  // 0-1.0
    };
    
    void add_peer(const PeerInfo& peer) {
        known_peers.push_back(peer);
        // Gossip to other peers
        broadcast_peer_info(peer);
    }
    
    std::vector<PeerInfo> get_nearby_peers(
        const UUID& exclude_id,
        int count = 10
    ) {
        // Sort by latency and reputation
        std::partial_sort(
            known_peers.begin(),
            known_peers.begin() + std::min(count, (int)known_peers.size()),
            known_peers.end(),
            [](const auto& a, const auto& b) {
                auto score_a = a.reputation / (1.0f + estimate_latency(a));
                auto score_b = b.reputation / (1.0f + estimate_latency(b));
                return score_a > score_b;
            }
        );
        
        std::vector<PeerInfo> result(
            known_peers.begin(),
            known_peers.begin() + std::min(count, (int)known_peers.size())
        );
        return result;
    }
};
```

---

## Latency Optimization

### Techniques

#### 1. Client-Side Prediction
```cpp
// Predict entity position before server confirms
Vec3 predicted_position = current_position + (velocity * dt);

// When server update arrives, smoothly reconcile
if (distance(server_position, predicted_position) > THRESHOLD) {
    // Major discrepancy, snap to server
    position = server_position;
} else {
    // Minor, smoothly interpolate
    position = lerp(position, server_position, 0.1f);
}
```

#### 2. Priority-Based Sending
```cpp
struct SendQueue {
    std::priority_queue<MessageFrame> messages;  // By priority
    
    void enqueue(const MessageFrame& frame) {
        messages.push(frame);
    }
    
    void flush(int bandwidth_available) {
        int bytes_sent = 0;
        
        while (!messages.empty() && bytes_sent < bandwidth_available) {
            auto frame = messages.top();
            messages.pop();
            
            int frame_size = frame.header.frame_size;
            send_frame(frame);
            bytes_sent += frame_size;
        }
    }
};
```

#### 3. Batching
```cpp
class MessageBatcher {
    std::vector<EntityUpdatePayload> pending_updates;
    uint64_t last_flush;
    
    void add_update(const EntityUpdatePayload& update) {
        pending_updates.push_back(update);
        
        if (pending_updates.size() >= BATCH_SIZE ||
            now() - last_flush > BATCH_TIMEOUT) {
            flush();
        }
    }
    
    void flush() {
        // Send all pending updates in one message
        auto frame = create_batch_message(pending_updates);
        send_message(frame);
        
        pending_updates.clear();
        last_flush = now();
    }
};
```

#### 4. Interest Management
```cpp
class InterestManager {
    // Only send updates for nearby entities
    constexpr float SEND_RADIUS = 100.0f;
    constexpr float DETAILED_RADIUS = 50.0f;
    
    void send_updates_for_player(const UUID& player_id, const Vec3& player_pos) {
        for (const auto& entity : world.entities) {
            float distance = vec3_distance(player_pos, entity.position);
            
            if (distance > SEND_RADIUS) {
                continue;  // Don't send
            }
            
            if (distance < DETAILED_RADIUS) {
                send_detailed_update(entity);      // 20 Hz
            } else {
                send_simplified_update(entity);    // 5 Hz
            }
        }
    }
};
```

### Performance Metrics

```cpp
struct NetworkMetrics {
    float avg_latency_ms;
    float p99_latency_ms;
    float packet_loss_rate;
    float bandwidth_usage_mbps;
    float compression_ratio;
    
    void log() {
        log_info(
            "Network: latency={:.1f}ms (p99={:.1f}ms), "
            "loss={:.2f}%, bw={:.2f}Mbps, compress={:.1f}%",
            avg_latency_ms, p99_latency_ms, packet_loss_rate * 100,
            bandwidth_usage_mbps, compression_ratio * 100
        );
    }
};
```

---

## Summary

Project ECHO's networking achieves:
✅ QUIC/HTTP3 for low-latency, multiplexed transport
✅ Custom binary protocol for efficient message passing
✅ FlatBuffers for zero-copy serialization
✅ Zstd/LZ4 compression (20-60% size reduction)
✅ P2P mesh networking with relay fallback
✅ Sub-100ms p99 latency through optimization
✅ Graceful degradation in poor network conditions
