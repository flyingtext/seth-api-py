# Seth Quantum Cloud - Gateway API Specification

**Version:** 1.0.0
**Base URL:** `https://sethquantum.net/api/v1`
**Protocol:** HTTPS
**Format:** JSON, SBF (Seth Binary Format)

---

## Table of Contents

1. [Authentication](#authentication)
2. [API Endpoints](#api-endpoints)
3. [Synchronous Execution API](#synchronous-execution-api)
4. [Asynchronous Execution API](#asynchronous-execution-api)
5. [SBF (Seth Binary Format)](#sbf-seth-binary-format)
6. [Quantum Circuit Structure](#quantum-circuit-structure)
7. [Circuit Samples](#circuit-samples)
8. [Error Handling](#error-handling)
9. [Usage Analytics](#usage-analytics)
10. [System Limits](#system-limits)

---

## Authentication

Seth Quantum Cloud supports two authentication methods.

### 1. API Key Authentication (Recommended)

All compute requests require an API Key.

**Header:**
```http
X-API-Key: sk_live_your_api_key_here
```

### 2. JWT Token Authentication

Used for Dashboard and management operations.

**Header:**
```http
Authorization: Bearer your_jwt_token_here
```

### Issuing an API Key

```bash
# 1. Register
curl -X POST https://sethquantum.net/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "your_username",
    "password": "your_password",
    "organization": "Your Organization"
  }'

# 2. Login to receive JWT token
curl -X POST https://sethquantum.net/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "your_username",
    "password": "your_password"
  }'

# 3. Create API Key
curl -X POST https://sethquantum.net/api/v1/keys \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My API Key",
    "description": "For production use"
  }'
```

**Response:**
```json
{
  "key_id": "key_abc123def456",
  "name": "My API Key",
  "api_key": "sk_live_xyz789abc123def456ghi789jkl012",
  "created_at": "2025-01-15T10:30:00Z",
  "is_active": true
}
```

⚠️ **Important:** The API Key is only displayed once upon creation. Store it securely.

---

## API Endpoints

### Authentication

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | User registration |
| POST | `/auth/login` | Login (JWT issuance) |
| GET | `/auth/me` | Current user information |

### API Key Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/keys` | List API Keys |
| POST | `/keys` | Create API Key |
| GET | `/keys/{key_id}` | Get API Key details |
| PUT | `/keys/{key_id}` | Update API Key (name, description) |
| DELETE | `/keys/{key_id}` | Delete API Key |
| POST | `/keys/{key_id}/regenerate` | Regenerate API Key |

### Quantum Circuit Execution

| Method | Endpoint | Description | Type |
|--------|----------|-------------|------|
| POST | `/compute/run` | Synchronous execution (JSON) | Sync |
| POST | `/compute/run/sbf` | Synchronous execution (Binary) | Sync |
| POST | `/compute/jobs` | Submit async job (JSON) | Async |
| POST | `/compute/jobs/sbf` | Submit async job (Binary) | Async |
| GET | `/compute/jobs/{job_id}` | Query job status | Async |
| GET | `/compute/jobs/{job_id}/result` | Retrieve job result | Async |
| DELETE | `/compute/jobs/{job_id}` | Cancel/delete job | Async |

### Usage Analytics

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/usage` | Usage statistics |
| GET | `/usage/logs` | Request logs |

---

## Synchronous Execution API

Execute fast circuits (<30s, <10 qubits recommended) immediately and receive results.

### POST `/compute/run`

**Request:**
```http
POST /api/v1/compute/run
X-API-Key: sk_live_your_api_key_here
Content-Type: application/json

{
  "circuit": {
    "name": "Bell State Circuit",
    "n_qubits": 2,
    "gates": [
      {
        "type": "H",
        "target": 0
      },
      {
        "type": "CNOT",
        "control": 0,
        "target": 1
      }
    ],
    "shots": 1000,
    "seed": 42
  }
}
```

**Response (Success):**
```json
{
  "circuit": {
    "name": "Bell State Circuit",
    "n_qubits": 2,
    "gates": [
      {
        "type": "H",
        "target": 0
      },
      {
        "type": "CNOT",
        "control": 0,
        "target": 1
      }
    ],
    "shots": 1000,
    "seed": 42
  },
  "result": {
    "counts": {
      "00": 497,
      "11": 503
    },
    "shots": 1000,
    "execution_time_ms": 23.45,
    "timestamp": "2025-01-15T10:35:12.345Z"
  },
  "metadata": {
    "server_version": "1.0.0",
    "backend": "seth_cpt_simulator"
  }
}
```

**Timeout:** 120 seconds (2 minutes)

---

## Asynchronous Execution API

Execute large-scale circuits requiring long execution time in the background.

### 1. Submit Job

**POST `/compute/jobs`**

```http
POST /api/v1/compute/jobs
X-API-Key: sk_live_your_api_key_here
Content-Type: application/json

{
  "circuit": {
    "name": "Large GHZ State",
    "n_qubits": 20,
    "gates": [
      {
        "type": "H",
        "target": 0
      },
      {
        "type": "CNOT",
        "control": 0,
        "target": 1
      },
      ...
    ],
    "shots": 10000,
    "seed": 42
  }
}
```

**Response:**
```json
{
  "job_id": "job_abc123def456ghi789",
  "status": "pending",
  "created_at": "2025-01-15T10:40:00Z",
  "circuit": {
    "name": "Large GHZ State",
    "n_qubits": 20,
    "shots": 10000
  }
}
```

### 2. Query Job Status

**GET `/compute/jobs/{job_id}`**

```http
GET /api/v1/compute/jobs/job_abc123def456ghi789
X-API-Key: sk_live_your_api_key_here
```

**Response (Running):**
```json
{
  "job_id": "job_abc123def456ghi789",
  "status": "running",
  "progress": 0.65,
  "created_at": "2025-01-15T10:40:00Z",
  "started_at": "2025-01-15T10:40:05Z",
  "circuit": {
    "name": "Large GHZ State",
    "n_qubits": 20,
    "shots": 10000
  }
}
```

**Status Values:**
- `pending`: Waiting
- `preparing`: Preparing
- `running`: Running
- `completed`: Completed
- `failed`: Failed
- `cancelled`: Cancelled

### 3. Retrieve Result

**GET `/compute/jobs/{job_id}/result`**

```http
GET /api/v1/compute/jobs/job_abc123def456ghi789/result
X-API-Key: sk_live_your_api_key_here
```

**Response:**
```json
{
  "job_id": "job_abc123def456ghi789",
  "status": "completed",
  "circuit": {
    "name": "Large GHZ State",
    "n_qubits": 20,
    "shots": 10000
  },
  "result": {
    "counts": {
      "00000000000000000000": 4987,
      "11111111111111111111": 5013
    },
    "shots": 10000,
    "execution_time_ms": 1234.56,
    "timestamp": "2025-01-15T10:45:30Z"
  },
  "created_at": "2025-01-15T10:40:00Z",
  "completed_at": "2025-01-15T10:45:30Z"
}
```

### 4. Cancel Job

**DELETE `/compute/jobs/{job_id}`**

```http
DELETE /api/v1/compute/jobs/job_abc123def456ghi789
X-API-Key: sk_live_your_api_key_here
```

---

## SBF (Seth Binary Format)

Binary format for high-performance quantum circuit transmission. Provides **up to 10x faster transmission** compared to JSON.

### When to Use SBF?

- ✅ Large-scale circuits (>100 gates)
- ✅ High throughput requirements
- ✅ Network bandwidth savings
- ✅ Production environments

### SBF Format Structure

```
┌─────────────────────────────────────────────────────────┐
│ Header (16 bytes)                                        │
├─────────────────────────────────────────────────────────┤
│ Magic Number (4 bytes): 0x53424600 ("SBF\0")           │
│ Version (2 bytes): 0x0001                               │
│ Flags (2 bytes): Reserved                               │
│ n_qubits (4 bytes): uint32                              │
│ n_gates (4 bytes): uint32                               │
├─────────────────────────────────────────────────────────┤
│ Circuit Name (variable)                                  │
├─────────────────────────────────────────────────────────┤
│ Name Length (2 bytes): uint16                           │
│ Name (UTF-8, max 255 bytes)                             │
├─────────────────────────────────────────────────────────┤
│ Gates Array (variable)                                   │
├─────────────────────────────────────────────────────────┤
│ For each gate:                                           │
│   Type (1 byte): enum                                    │
│   Target (4 bytes): uint32                               │
│   Control (4 bytes): uint32 or 0xFFFFFFFF if none       │
│   Multi-controls count (4 bytes): uint32                │
│   Multi-controls array (4 bytes each): uint32[]         │
│   Parameters count (1 byte): uint8                      │
│   Parameters (8 bytes each): float64[]                  │
├─────────────────────────────────────────────────────────┤
│ Footer (12 bytes)                                        │
├─────────────────────────────────────────────────────────┤
│ Shots (4 bytes): uint32                                 │
│ Seed (4 bytes): int32 or 0xFFFFFFFF if none             │
│ Checksum (4 bytes): CRC32                               │
└─────────────────────────────────────────────────────────┘
```

### Gate Type Enumeration

| Type | Value | Description |
|------|-------|-------------|
| I | 0x00 | Identity |
| X | 0x01 | Pauli-X (NOT) |
| Y | 0x02 | Pauli-Y |
| Z | 0x03 | Pauli-Z |
| H | 0x04 | Hadamard |
| S | 0x05 | S gate (√Z) |
| T | 0x06 | T gate (√S) |
| RX | 0x10 | Rotation-X (1 param) |
| RY | 0x11 | Rotation-Y (1 param) |
| RZ | 0x12 | Rotation-Z (1 param) |
| CNOT | 0x20 | Controlled-NOT |
| CZ | 0x21 | Controlled-Z |
| SWAP | 0x22 | SWAP |
| TOFFOLI | 0x30 | Toffoli (CCX) |
| FREDKIN | 0x31 | Fredkin (CSWAP) |

### SBF Usage Example (Python)

```python
import struct
import requests
from io import BytesIO

def encode_sbf_circuit(circuit):
    """Encode quantum circuit to SBF format"""
    buf = BytesIO()

    # Header
    buf.write(struct.pack('<I', 0x53424600))  # Magic: "SBF\0"
    buf.write(struct.pack('<H', 1))           # Version: 1
    buf.write(struct.pack('<H', 0))           # Flags: 0
    buf.write(struct.pack('<I', circuit['n_qubits']))
    buf.write(struct.pack('<I', len(circuit['gates'])))

    # Circuit name
    name = circuit.get('name', '').encode('utf-8')[:255]
    buf.write(struct.pack('<H', len(name)))
    buf.write(name)

    # Gates
    gate_types = {
        'I': 0x00, 'X': 0x01, 'Y': 0x02, 'Z': 0x03,
        'H': 0x04, 'S': 0x05, 'T': 0x06,
        'RX': 0x10, 'RY': 0x11, 'RZ': 0x12,
        'CNOT': 0x20, 'CZ': 0x21, 'SWAP': 0x22,
        'TOFFOLI': 0x30, 'FREDKIN': 0x31
    }

    for gate in circuit['gates']:
        # Gate type
        buf.write(struct.pack('<B', gate_types[gate['type']]))

        # Target
        buf.write(struct.pack('<I', gate['target']))

        # Control (0xFFFFFFFF if none)
        control = gate.get('control', 0xFFFFFFFF)
        buf.write(struct.pack('<I', control if control != -1 else 0xFFFFFFFF))

        # Multi-controls
        controls = gate.get('controls', [])
        buf.write(struct.pack('<I', len(controls)))
        for ctrl in controls:
            buf.write(struct.pack('<I', ctrl))

        # Parameters
        params = gate.get('params', [])
        buf.write(struct.pack('<B', len(params)))
        for param in params:
            buf.write(struct.pack('<d', param))

    # Footer
    buf.write(struct.pack('<I', circuit.get('shots', 1000)))
    seed = circuit.get('seed', -1)
    buf.write(struct.pack('<i', seed if seed != -1 else 0xFFFFFFFF))

    # CRC32 checksum
    data = buf.getvalue()
    import zlib
    checksum = zlib.crc32(data) & 0xFFFFFFFF
    buf.write(struct.pack('<I', checksum))

    return buf.getvalue()

# Usage example
circuit = {
    "name": "Bell State",
    "n_qubits": 2,
    "gates": [
        {"type": "H", "target": 0},
        {"type": "CNOT", "control": 0, "target": 1}
    ],
    "shots": 1000,
    "seed": 42
}

sbf_data = encode_sbf_circuit(circuit)

# Synchronous execution with SBF
response = requests.post(
    'https://sethquantum.net/api/v1/compute/run/sbf',
    headers={'X-API-Key': 'sk_live_your_api_key'},
    data=sbf_data,
    headers={'Content-Type': 'application/octet-stream'}
)

result = response.json()
print(result['result']['counts'])
```

### SBF Response

Responses to SBF requests are **always in JSON format** for flexibility and compatibility of result data.

---

## Quantum Circuit Structure

### Circuit Object

```json
{
  "name": "Circuit Name",        // Circuit name (optional)
  "n_qubits": 2,                 // Number of qubits (required)
  "gates": [...],                // Gates array (required)
  "shots": 1000,                 // Number of measurement shots (optional, default: 10000)
  "seed": 42                     // Random seed (optional)
}
```

### Gate Object

**Single Qubit Gate:**
```json
{
  "type": "H",                   // Gate type
  "target": 0                    // Target qubit
}
```

**Parameterized Gate:**
```json
{
  "type": "RX",                  // Rotation-X
  "target": 0,
  "params": [1.5708]             // π/2 rotation
}
```

**Controlled Gate:**
```json
{
  "type": "CNOT",
  "control": 0,                  // Control qubit
  "target": 1                    // Target qubit
}
```

**Multi-Controlled Gate:**
```json
{
  "type": "TOFFOLI",
  "controls": [0, 1],            // Control qubits
  "target": 2                    // Target qubit
}
```

### Supported Gates

| Gate | Type | Description | Parameters |
|------|------|-------------|------------|
| **Single Qubit** |
| Identity | I | Identity gate | - |
| Pauli-X | X | NOT gate | - |
| Pauli-Y | Y | Y rotation | - |
| Pauli-Z | Z | Z rotation | - |
| Hadamard | H | Create superposition | - |
| S | S | √Z gate | - |
| T | T | √S gate | - |
| **Rotation Gates** |
| Rotation-X | RX | X-axis rotation | theta (radians) |
| Rotation-Y | RY | Y-axis rotation | theta (radians) |
| Rotation-Z | RZ | Z-axis rotation | theta (radians) |
| **2-Qubit Gates** |
| CNOT | CNOT | Controlled-NOT | - |
| CZ | CZ | Controlled-Z | - |
| SWAP | SWAP | Qubit swap | - |
| **Multi-Controlled Gates** |
| Toffoli | TOFFOLI | CCX (2 controls) | - |
| Fredkin | FREDKIN | CSWAP | - |

---

## Circuit Samples

### 1. Bell State

Creates the simplest quantum entanglement state.

```json
{
  "circuit": {
    "name": "Bell State |Φ+⟩",
    "n_qubits": 2,
    "gates": [
      {"type": "H", "target": 0},
      {"type": "CNOT", "control": 0, "target": 1}
    ],
    "shots": 1000,
    "seed": 42
  }
}
```

**Expected Result:** `|00⟩` and `|11⟩` each approximately 50% probability

### 2. GHZ State (3-qubit)

Maximum entanglement state of 3 qubits.

```json
{
  "circuit": {
    "name": "GHZ State",
    "n_qubits": 3,
    "gates": [
      {"type": "H", "target": 0},
      {"type": "CNOT", "control": 0, "target": 1},
      {"type": "CNOT", "control": 0, "target": 2}
    ],
    "shots": 2000
  }
}
```

**Expected Result:** `|000⟩` and `|111⟩` each approximately 50% probability

### 3. Quantum Fourier Transform (QFT)

3-qubit Quantum Fourier Transform.

```json
{
  "circuit": {
    "name": "3-Qubit QFT",
    "n_qubits": 3,
    "gates": [
      {"type": "H", "target": 0},
      {"type": "RZ", "target": 0, "params": [0.7854]},
      {"type": "CNOT", "control": 1, "target": 0},
      {"type": "RZ", "target": 0, "params": [0.3927]},
      {"type": "H", "target": 1},
      {"type": "RZ", "target": 1, "params": [0.7854]},
      {"type": "CNOT", "control": 2, "target": 1},
      {"type": "H", "target": 2},
      {"type": "SWAP", "control": 0, "target": 2}
    ],
    "shots": 5000
  }
}
```

### 4. Grover's Algorithm (2-qubit)

Simple 2-qubit Grover search algorithm.

```json
{
  "circuit": {
    "name": "Grover Search (2-qubit)",
    "n_qubits": 2,
    "gates": [
      {"type": "H", "target": 0},
      {"type": "H", "target": 1},
      {"type": "Z", "target": 1},
      {"type": "CZ", "control": 0, "target": 1},
      {"type": "H", "target": 0},
      {"type": "H", "target": 1},
      {"type": "X", "target": 0},
      {"type": "X", "target": 1},
      {"type": "CZ", "control": 0, "target": 1},
      {"type": "X", "target": 0},
      {"type": "X", "target": 1},
      {"type": "H", "target": 0},
      {"type": "H", "target": 1}
    ],
    "shots": 1000
  }
}
```

### 5. Toffoli Gate Test

Multi-controlled gate example.

```json
{
  "circuit": {
    "name": "Toffoli Test",
    "n_qubits": 3,
    "gates": [
      {"type": "X", "target": 0},
      {"type": "X", "target": 1},
      {"type": "TOFFOLI", "controls": [0, 1], "target": 2}
    ],
    "shots": 1000
  }
}
```

**Expected Result:** `|111⟩` 100% (input: `|110⟩`)

### 6. Parameterized Circuit

Variational quantum circuit using rotation gates.

```json
{
  "circuit": {
    "name": "Parameterized Circuit",
    "n_qubits": 2,
    "gates": [
      {"type": "RY", "target": 0, "params": [0.5236]},
      {"type": "RY", "target": 1, "params": [1.0472]},
      {"type": "CNOT", "control": 0, "target": 1},
      {"type": "RZ", "target": 0, "params": [0.7854]},
      {"type": "RZ", "target": 1, "params": [1.5708]}
    ],
    "shots": 10000
  }
}
```

---

## Error Handling

### Error Response Structure

```json
{
  "error": {
    "code": "error_code",
    "message": "Human-readable error message",
    "details": "Additional details (optional)"
  }
}
```

### Error Codes

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `invalid_request` | Invalid request format |
| 400 | `invalid_circuit` | Invalid circuit structure |
| 400 | `qubit_limit_exceeded` | Qubit limit exceeded |
| 401 | `unauthorized` | Authentication failed |
| 401 | `invalid_api_key` | Invalid API Key |
| 403 | `forbidden` | Permission denied |
| 404 | `not_found` | Resource not found |
| 404 | `job_not_found` | Job not found |
| 429 | `rate_limit_exceeded` | Rate limit exceeded |
| 500 | `internal_error` | Internal server error |
| 503 | `queue_full` | Job queue saturated |
| 503 | `compute_unavailable` | Compute server not responding |

### Error Examples

**Qubit Limit Exceeded:**
```json
{
  "error": {
    "code": "invalid_request",
    "message": "n_qubits exceeds your limit of 1,000 qubits (alpha tier)",
    "details": "Requested: 2000, Allowed: 1000"
  }
}
```

**Invalid Gate:**
```json
{
  "error": {
    "code": "invalid_circuit",
    "message": "Gate 5: target qubit index out of range",
    "details": "Target qubit 10 exceeds circuit size (n_qubits=5)"
  }
}
```

**Queue Saturated:**
```json
{
  "error": {
    "code": "queue_full",
    "message": "Job queue is full. Please try again later.",
    "details": "Current queue: 1024/1024"
  }
}
```

---

## Usage Analytics

### GET `/usage`

Query usage statistics by API Key.

```http
GET /api/v1/usage?key_id=key_abc123&days=7
Authorization: Bearer YOUR_JWT_TOKEN
```

**Query Parameters:**
- `key_id` (optional): Filter by specific API Key
- `days` (optional): Query period (default: 30 days)

**Response:**
```json
{
  "summary": {
    "total_requests": 1523,
    "sync_requests": 1200,
    "async_jobs": 323,
    "total_qubits_processed": 45123,
    "avg_execution_time_ms": 234.5
  },
  "by_key": [
    {
      "key_id": "key_abc123",
      "key_name": "Production Key",
      "requests": 1000,
      "jobs": 200,
      "last_used": "2025-01-15T10:30:00Z"
    }
  ],
  "timeline": [
    {
      "date": "2025-01-15",
      "requests": 150,
      "jobs": 30
    }
  ]
}
```

### GET `/usage/logs`

Query detailed request logs.

```http
GET /api/v1/usage/logs?limit=50&offset=0
Authorization: Bearer YOUR_JWT_TOKEN
```

**Response:**
```json
{
  "logs": [
    {
      "request_id": "req_xyz789",
      "timestamp": "2025-01-15T10:35:12Z",
      "request_type": "sync",
      "circuit": {
        "name": "Bell State",
        "n_qubits": 2,
        "gates_count": 2,
        "shots": 1000
      },
      "status_code": 200,
      "execution_time_ms": 23.45
    }
  ],
  "pagination": {
    "limit": 50,
    "offset": 0,
    "total": 1523
  }
}
```

---

## System Limits

### Alpha Version Limits

| Item | Limit | Description |
|------|-------|-------------|
| **Qubits** | 1,000 | Maximum qubits per user |
| **Gates** | 67,108,864 | Maximum gates per circuit |
| **Shots** | 67,108,864 | Maximum measurements |
| **Concurrent Jobs** | Unlimited | Async job submissions |
| **Request Size** | 1,000 MB | HTTP request size |

### System Maximum Limits

| Item | Limit | Description |
|------|-------|-------------|
| **Max Qubits** | 67,108,864 | System absolute limit |
| **Max Controls** | 65,536 | Multi-control gates |
| **Queue Capacity** | 1,024 | Concurrent processing jobs |
| **Job Retention** | 1 hour | Auto-deletion after completion |

### Performance Guidelines

| Circuit Size | Recommended API | Expected Time |
|--------------|-----------------|---------------|
| < 10 qubits, < 50 gates | Sync API | < 5 seconds |
| 10-20 qubits, < 100 gates | Sync/Async | 5-30 seconds |
| 20-100 qubits | Async API | 30s-10 minutes |
| 100-1000 qubits | Async API | 10min-2 hours |

---

## Quick Start Examples

### Python

```python
import requests
import json

API_KEY = "sk_live_your_api_key_here"
BASE_URL = "https://sethquantum.net/api/v1"

# Bell state circuit
circuit = {
    "circuit": {
        "name": "Bell State",
        "n_qubits": 2,
        "gates": [
            {"type": "H", "target": 0},
            {"type": "CNOT", "control": 0, "target": 1}
        ],
        "shots": 1000
    }
}

# Synchronous execution
response = requests.post(
    f"{BASE_URL}/compute/run",
    headers={
        "X-API-Key": API_KEY,
        "Content-Type": "application/json"
    },
    json=circuit
)

result = response.json()
print("Counts:", result['result']['counts'])
print("Time:", result['result']['execution_time_ms'], "ms")
```

### JavaScript (Node.js)

```javascript
const axios = require('axios');

const API_KEY = 'sk_live_your_api_key_here';
const BASE_URL = 'https://sethquantum.net/api/v1';

const circuit = {
  circuit: {
    name: 'Bell State',
    n_qubits: 2,
    gates: [
      { type: 'H', target: 0 },
      { type: 'CNOT', control: 0, target: 1 }
    ],
    shots: 1000
  }
};

// Synchronous execution
axios.post(`${BASE_URL}/compute/run`, circuit, {
  headers: {
    'X-API-Key': API_KEY,
    'Content-Type': 'application/json'
  }
})
.then(response => {
  console.log('Counts:', response.data.result.counts);
  console.log('Time:', response.data.result.execution_time_ms, 'ms');
})
.catch(error => {
  console.error('Error:', error.response.data.error);
});
```

### cURL

```bash
curl -X POST https://sethquantum.net/api/v1/compute/run \
  -H "X-API-Key: sk_live_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "circuit": {
      "name": "Bell State",
      "n_qubits": 2,
      "gates": [
        {"type": "H", "target": 0},
        {"type": "CNOT", "control": 0, "target": 1}
      ],
      "shots": 1000
    }
  }'
```

---

## Support

- **Website**: https://sethquantum.net
- **Dashboard**: https://sethquantum.net/dashboard
- **Documentation**: https://docs.sethquantum.net
- **Email**: flyingtext@nate.com

---

**Seth Quantum Cloud Gateway API v1.0.0**
*High-Performance Quantum Circuit Simulation as a Service*

© 2025 flyingtext / 윤지현. All rights reserved.
