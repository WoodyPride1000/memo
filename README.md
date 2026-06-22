# OpenMesh Lightweight Telemetry Protocol (OLTP)

## Event-Driven Real-Time Extension Specification

### 1. Design Philosophy

OpenMesh is designed as an ultra-low-power Delay Tolerant Network (DTN) optimized for large-scale IoT deployments operating in areas lacking communication infrastructure.

The protocol intentionally avoids maintaining continuous real-time connectivity.

Instead, the network operates in two modes:

* Normal Monitoring Mode
* Event-Driven Real-Time Mode

This architecture minimizes airtime usage, battery consumption, and network congestion while preserving the ability to rapidly react to critical events.

---

## 2. Operating Modes

### 2.1 Normal Monitoring Mode

Purpose:

* Position reporting
* Health monitoring
* Presence monitoring
* Environmental sensing

Characteristics:

* Transmission interval: configurable (default 1 hour)
* Minimum airtime
* Maximum battery life
* DTN-based store-and-forward operation

Flags:

REALTIME = 0

Example:

Cow #1024
Latitude
Longitude
Timestamp
Battery Status

Transmission frequency:

1 packet/hour

---

### 2.2 Event-Driven Real-Time Mode

Activated only when exceptional conditions occur.

Examples:

* Geofence violation
* Immobility detection
* Fall detection
* Theft detection
* Emergency button activation
* Sensor threshold exceeded

Flags:

REALTIME = 1

When activated:

* Reporting interval reduced
* Forwarding probability increased
* Relay priority increased
* TTL increased

This temporarily transforms the DTN into a high-delivery emergency network.

---

## 3. Packet Header Extension

### FLAGS Field

| Bit | Name        | Description                      |
| --- | ----------- | -------------------------------- |
| 7   | AUTH        | Authentication enabled           |
| 6   | COMPRESS    | Payload compressed               |
| 5   | ENCRYPT     | Payload encrypted                |
| 4   | REALTIME    | Event-driven emergency mode      |
| 3   | ACK_REQUEST | Explicit acknowledgement request |
| 2-0 | Reserved    | Future expansion                 |

---

## 4. Forwarding Policy

### Normal Packet

REALTIME = 0

Recommended parameters:

Forward Probability = 0.20

TTL = 3

Objective:

Minimize network load.

---

### Emergency Packet

REALTIME = 1

Recommended parameters:

Forward Probability = 1.00

TTL = 8

Objective:

Maximize delivery probability.

---

## 5. Density-Aware Adaptive Relay

Each node maintains a local neighbor estimate.

No dedicated HELLO packets are required.

Density estimation is derived from ordinary telemetry packets.

### Local Density Index (LDI)

Definition:

LDI = Unique Node IDs received during previous observation window

Example:

LDI < 10
Sparse region

10 ≤ LDI ≤ 50
Normal region

LDI > 50
Dense region

---

### Relay Adaptation

Sparse region:

Forward Probability = 1.0

Normal region:

Forward Probability = 0.5

Dense region:

Forward Probability = 0.1

This mechanism automatically suppresses broadcast storms without maintaining routing tables.

---

## 6. Location-Aware Forwarding

Each packet contains:

* Sender Position
* Receiver Position
* Optional Gateway Position

Relay nodes calculate forwarding usefulness.

Example scoring function:

Score =
α × Gateway Progress
− β × Local Density
− γ × Packet Age

Packets with higher scores are preferentially retransmitted.

---

## 7. Authentication and Security

Authentication mechanism:

HMAC-SHA256 (truncated 16 bytes)

Input:

VER || UID || LAT || LON || TS || SEQ || SALT

Output:

16-byte authentication tag

Benefits:

* Low computational cost
* Replay attack protection
* Suitable for ESP32-class devices
* Minimal airtime overhead

---

## 8. Replay Protection

Three-layer protection:

1. Timestamp Window

Accept only packets within ±30 seconds.

2. Sequence Validation

Monotonically increasing sequence numbers.

3. Salt Validation

Recent salt values stored using Bloom Filter.

Duplicate salts rejected.

---

## 9. Network Philosophy

OpenMesh intentionally avoids:

* Routing tables
* Neighbor tables
* Link-state flooding
* OLSR-style topology maintenance

Instead, it relies on:

* Opportunistic forwarding
* Density-aware relay decisions
* Position-assisted forwarding
* Event-driven real-time escalation

This enables operation at extremely low power while remaining scalable to very large deployments.

---

## 10. Target Applications

Primary Markets:

* Livestock monitoring
* Ranch management
* Wildlife tracking
* Remote agriculture

Future Markets:

* Asset tracking
* Industrial IoT
* Disaster recovery networks
* Human safety monitoring
* Temporary tactical communication systems

OpenMesh is intended to become a lightweight, infrastructure-independent IoT transport layer for large-scale, delay-tolerant sensing networks.
