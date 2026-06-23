# OpenMesh Lightweight Telemetry Protocol (OLTP)

## Event-Driven Real-Time Extension Specification

---

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
| 5   | ENCRYPT     | Payload encrypted (AES-128-GCM)  |
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

Relay nodes calculate forwarding usefulness using the following scoring function:

```
Score = α × GatewayProgress − β × LocalDensity − γ × PacketAge
```

### Recommended Default Coefficients

| Coefficient | Default | Description                        | Tunable Range |
| ----------- | ------- | ---------------------------------- | ------------- |
| α           | 0.6     | Weight for gateway progress        | 0.4 – 0.8    |
| β           | 0.3     | Penalty for local node density     | 0.1 – 0.5    |
| γ           | 0.1     | Penalty for packet age             | 0.05 – 0.3   |

These defaults are optimized for sparse rural deployments (livestock, wildlife tracking).
Implementations MAY adjust coefficients within the tunable range to suit local conditions.
Implementations MUST document any deviation from the defaults.

Packets with higher scores are preferentially retransmitted.

---

## 7. Authentication and Security

### 7.1 Authentication Mechanism

Authentication uses HMAC-SHA256 with a 128-bit (16-byte) truncated output tag,
compliant with RFC 2104 (HMAC) and NIST SP 800-107 (truncation guidance).

Input:

```
VER || UID || LAT || LON || TS || SEQ || SALT
```

Output:

16-byte authentication tag (HMAC-SHA256/128)

Benefits:

* Low computational cost — suitable for ESP32-class devices
* Replay attack protection (in combination with Section 8)
* Minimal airtime overhead (16 bytes per packet)

> **Note:** Implementations requiring higher security assurance (e.g., asset tracking,
> human safety monitoring) SHOULD use the full 32-byte HMAC-SHA256 output.
> The 16-byte truncation reduces the birthday-attack bound to 2⁶⁴ operations,
> which is acceptable for short-lived IoT sessions but should be reviewed for
> long-term deployments exceeding several years.

---

### 7.2 Encryption (ENCRYPT flag)

When `FLAGS.ENCRYPT = 1`, the following fields MUST be encrypted:

Encrypted fields: `LAT || LON || TS`

Algorithm: **AES-128-GCM**

* Key: derived from the per-device SecretKey (see Section 7.3)
* Nonce: 12 bytes, constructed as `SEQ (4B) || SALT (8B)`
* Authentication Tag: 16 bytes (GCM built-in, replaces HMAC when ENCRYPT=1)

> AES-128-GCM provides both confidentiality and authenticated integrity in a single
> pass, and is natively supported on ESP32 via the hardware AES accelerator.
> When ENCRYPT=1, the GCM authentication tag serves as the authentication tag
> and the AUTH flag (bit 7) is implicitly set.

---

### 7.3 Key Management

#### Key Generation

Each device is provisioned with a unique **SecretKey** (128-bit) during manufacture
or secure onboarding. Keys MUST be generated using a cryptographically secure
random number generator (CSPRNG).

Per-device subkeys are derived using HKDF (RFC 5869):

```
DeviceKey = HKDF-SHA256(MasterKey, salt=UID, info="OLTP-v2")
```

This ensures that compromise of one device key does not expose other devices.

#### Key Distribution

* SecretKey exchange MUST occur over a mutually authenticated TLS 1.3 channel.
* Keys MUST NOT be transmitted in plaintext.
* The registration server MUST verify device identity before issuing a key
  (e.g., hardware serial number, X.509 device certificate, or pre-shared token).

#### Key Rotation

| Condition                        | Action                          |
| -------------------------------- | ------------------------------- |
| Scheduled (default: every 90 days) | Issue new DeviceKey via secure channel |
| SEQ number rollover (2³² packets) | Mandatory key renegotiation     |
| Suspected compromise             | Immediate revocation + rekey    |

#### Key Revocation (REVOKE command)

The server MAY issue a REVOKE command to invalidate a device key:

```
REVOKE packet: VER=2 | FLAGS.AUTH=1 | UID | REVOKE_TOKEN | HMAC
```

Upon receiving a valid REVOKE packet:

1. The device MUST cease transmission immediately.
2. The device MUST await a new key via the secure onboarding channel.
3. Relay nodes MUST drop all subsequent packets from the revoked UID.

#### Key Storage

* Device-side: keys MUST be stored in a secure element or flash region protected
  against readout (e.g., ESP32 NVS encryption, eFuse-based key storage).
* Server-side: keys MUST be stored in an HSM or equivalent KMS.

---

## 8. Replay Protection

Three-layer protection:

### Layer 1 — Timestamp Window

Accept only packets within ±30 seconds of the receiver's current time.

Packets outside this window MUST be silently discarded.

### Layer 2 — Sequence Number Validation

SEQ is a **32-bit (4-byte) unsigned integer**, monotonically increasing per device.

* Valid range: 0x00000000 – 0xFFFFFFFF (~4.3 billion packets)
* At 1 packet/second continuous transmission: rollover occurs after ~136 years.
* Relay nodes MUST reject packets with SEQ ≤ last accepted SEQ for that UID.
* On SEQ rollover, the device MUST perform mandatory key renegotiation (see Section 7.3).

### Layer 3 — Salt Validation

SALT is an **8-byte (64-bit) cryptographically random value**, generated fresh for
each packet using a CSPRNG.

Duplicate detection uses a **Bloom Filter** maintained per UID:

* Target false-positive rate: < 0.1%
* Estimated memory: ~2 KB per 10,000 recent salts
* Collision probability at 1 packet/second: negligible (birthday bound ≈ 2³²)

Packets with a salt value matching a recent entry MUST be silently discarded.

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

---

## Appendix A — Packet Size Summary

| Configuration          | Size     | 10 kbps transmission time |
| ---------------------- | -------- | ------------------------- |
| AUTH only (HMAC-16B)   | 36 bytes | ~29 ms                    |
| AUTH + ENCRYPT (GCM)   | 52 bytes | ~42 ms                    |
| Full 32B HMAC (no GCM) | 52 bytes | ~42 ms                    |

All configurations are practical for 10 kbps low-power radio links.

---

## Appendix B — Changelog

| Version | Change                                                                 |
| ------- | ---------------------------------------------------------------------- |
| 2.0     | Added Section 7.2 (AES-128-GCM encryption spec)                        |
| 2.0     | Added Section 7.3 (Key management, rotation, revocation)               |
| 2.0     | SEQ field expanded to 32-bit; rollover policy defined (Section 8)      |
| 2.0     | SALT expanded to 8 bytes; Bloom Filter spec updated (Section 8)        |
| 2.0     | Score function default coefficients added (Section 6)                  |
| 2.0     | HMAC truncation rationale and security note added (Section 7.1)        |
| 1.0     | Initial release                                                        |
