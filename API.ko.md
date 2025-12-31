# Seth Quantum Cloud - Gateway API 규격서

**Version:** 1.0.0
**Base URL:** `https://sethquantum.net/api/v1`
**Protocol:** HTTPS
**Format:** JSON, SBF (Seth Binary Format)

---

## 목차

1. [인증](#인증)
2. [API 엔드포인트](#api-엔드포인트)
3. [동기 실행 API](#동기-실행-api)
4. [비동기 실행 API](#비동기-실행-api)
5. [SBF (Seth Binary Format)](#sbf-seth-binary-format)
6. [양자 회로 구조](#양자-회로-구조)
7. [회로 샘플](#회로-샘플)
8. [에러 처리](#에러-처리)
9. [사용량 조회](#사용량-조회)
10. [시스템 제한](#시스템-제한)

---

## 인증

Seth Quantum Cloud는 두 가지 인증 방식을 지원합니다.

### 1. API Key 인증 (권장)

모든 compute 요청에는 API Key가 필요합니다.

**Header:**
```http
X-API-Key: sk_live_your_api_key_here
```

### 2. JWT 토큰 인증

Dashboard 및 관리 작업에 사용됩니다.

**Header:**
```http
Authorization: Bearer your_jwt_token_here
```

### API Key 발급

```bash
# 1. 회원가입
curl -X POST https://sethquantum.net/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "your_username",
    "password": "your_password",
    "organization": "Your Organization"
  }'

# 2. 로그인하여 JWT 토큰 받기
curl -X POST https://sethquantum.net/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "your_username",
    "password": "your_password"
  }'

# 3. API Key 생성
curl -X POST https://sethquantum.net/api/v1/keys \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My API Key",
    "description": "For production use"
  }'
```

**응답:**
```json
{
  "key_id": "key_abc123def456",
  "name": "My API Key",
  "api_key": "sk_live_xyz789abc123def456ghi789jkl012",
  "created_at": "2025-01-15T10:30:00Z",
  "is_active": true
}
```

⚠️ **중요:** API Key는 생성 시 한 번만 표시됩니다. 안전하게 보관하세요.

---

## API 엔드포인트

### 인증 관련

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | 회원가입 |
| POST | `/auth/login` | 로그인 (JWT 발급) |
| GET | `/auth/me` | 현재 사용자 정보 |

### API Key 관리

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/keys` | API Key 목록 조회 |
| POST | `/keys` | API Key 생성 |
| GET | `/keys/{key_id}` | API Key 상세 조회 |
| PUT | `/keys/{key_id}` | API Key 수정 (이름, 설명) |
| DELETE | `/keys/{key_id}` | API Key 삭제 |
| POST | `/keys/{key_id}/regenerate` | API Key 재발급 |

### 양자 회로 실행

| Method | Endpoint | Description | Type |
|--------|----------|-------------|------|
| POST | `/compute/run` | 동기 실행 (JSON) | Sync |
| POST | `/compute/run/sbf` | 동기 실행 (Binary) | Sync |
| POST | `/compute/jobs` | 비동기 작업 제출 (JSON) | Async |
| POST | `/compute/jobs/sbf` | 비동기 작업 제출 (Binary) | Async |
| GET | `/compute/jobs/{job_id}` | 작업 상태 조회 | Async |
| GET | `/compute/jobs/{job_id}/result` | 작업 결과 조회 | Async |
| DELETE | `/compute/jobs/{job_id}` | 작업 취소/삭제 | Async |

### 사용량 조회

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/usage` | 사용량 통계 |
| GET | `/usage/logs` | 요청 로그 조회 |

---

## 동기 실행 API

빠른 회로(<30초, <10 큐빗 권장)를 즉시 실행하고 결과를 받습니다.

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

**Response (성공):**
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

**타임아웃:** 120초 (2분)

---

## 비동기 실행 API

긴 실행 시간이 필요한 대규모 회로를 백그라운드에서 실행합니다.

### 1. 작업 제출

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

### 2. 작업 상태 조회

**GET `/compute/jobs/{job_id}`**

```http
GET /api/v1/compute/jobs/job_abc123def456ghi789
X-API-Key: sk_live_your_api_key_here
```

**Response (실행 중):**
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

**상태 값:**
- `pending`: 대기 중
- `preparing`: 준비 중
- `running`: 실행 중
- `completed`: 완료
- `failed`: 실패
- `cancelled`: 취소됨

### 3. 결과 조회

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

### 4. 작업 취소

**DELETE `/compute/jobs/{job_id}`**

```http
DELETE /api/v1/compute/jobs/job_abc123def456ghi789
X-API-Key: sk_live_your_api_key_here
```

---

## SBF (Seth Binary Format)

고성능 양자 회로 전송을 위한 바이너리 포맷입니다. JSON 대비 **최대 10배 빠른 전송 속도**를 제공합니다.

### 언제 SBF를 사용하나요?

- ✅ 대규모 회로 (>100 게이트)
- ✅ 높은 처리량이 필요한 경우
- ✅ 네트워크 대역폭 절약
- ✅ 프로덕션 환경

### SBF 포맷 구조

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

### Gate Type 열거형

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

### SBF 사용 예제 (Python)

```python
import struct
import requests
from io import BytesIO

def encode_sbf_circuit(circuit):
    """양자 회로를 SBF 포맷으로 인코딩"""
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

# 사용 예제
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

# SBF로 동기 실행
response = requests.post(
    'https://sethquantum.net/api/v1/compute/run/sbf',
    headers={'X-API-Key': 'sk_live_your_api_key'},
    data=sbf_data,
    headers={'Content-Type': 'application/octet-stream'}
)

result = response.json()
print(result['result']['counts'])
```

### SBF 응답

SBF 요청에 대한 응답은 **항상 JSON 형식**으로 반환됩니다. 이는 결과 데이터의 유연성과 호환성을 위한 것입니다.

---

## 양자 회로 구조

### Circuit 객체

```json
{
  "name": "Circuit Name",        // 회로 이름 (선택)
  "n_qubits": 2,                 // 큐빗 수 (필수)
  "gates": [...],                // 게이트 배열 (필수)
  "shots": 1000,                 // 측정 샷 수 (선택, 기본: 10000)
  "seed": 42                     // 랜덤 시드 (선택)
}
```

### Gate 객체

**단일 큐빗 게이트:**
```json
{
  "type": "H",                   // 게이트 타입
  "target": 0                    // 대상 큐빗
}
```

**파라미터가 있는 게이트:**
```json
{
  "type": "RX",                  // Rotation-X
  "target": 0,
  "params": [1.5708]             // π/2 회전
}
```

**제어 게이트:**
```json
{
  "type": "CNOT",
  "control": 0,                  // 제어 큐빗
  "target": 1                    // 대상 큐빗
}
```

**다중 제어 게이트:**
```json
{
  "type": "TOFFOLI",
  "controls": [0, 1],            // 제어 큐빗들
  "target": 2                    // 대상 큐빗
}
```

### 지원 게이트

| 게이트 | 타입 | 설명 | 파라미터 |
|--------|------|------|----------|
| **단일 큐빗** |
| Identity | I | 항등 게이트 | - |
| Pauli-X | X | NOT 게이트 | - |
| Pauli-Y | Y | Y 회전 | - |
| Pauli-Z | Z | Z 회전 | - |
| Hadamard | H | 중첩 생성 | - |
| S | S | √Z 게이트 | - |
| T | T | √S 게이트 | - |
| **회전 게이트** |
| Rotation-X | RX | X축 회전 | theta (라디안) |
| Rotation-Y | RY | Y축 회전 | theta (라디안) |
| Rotation-Z | RZ | Z축 회전 | theta (라디안) |
| **2큐빗 게이트** |
| CNOT | CNOT | Controlled-NOT | - |
| CZ | CZ | Controlled-Z | - |
| SWAP | SWAP | 큐빗 교환 | - |
| **다중 제어 게이트** |
| Toffoli | TOFFOLI | CCX (2 controls) | - |
| Fredkin | FREDKIN | CSWAP | - |

---

## 회로 샘플

### 1. Bell State (벨 상태)

가장 간단한 양자 얽힘 상태를 생성합니다.

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

**예상 결과:** `|00⟩`과 `|11⟩`이 각각 약 50% 확률

### 2. GHZ State (3큐빗)

3개 큐빗의 최대 얽힘 상태입니다.

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

**예상 결과:** `|000⟩`과 `|111⟩`이 각각 약 50% 확률

### 3. Quantum Fourier Transform (QFT)

3큐빗 양자 푸리에 변환입니다.

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

### 4. Grover's Algorithm (2큐빗)

간단한 2큐빗 Grover 검색 알고리즘입니다.

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

### 5. Toffoli Gate 테스트

다중 제어 게이트 예제입니다.

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

**예상 결과:** `|111⟩` 100% (입력: `|110⟩`)

### 6. 파라미터화된 회로

회전 게이트를 사용한 변분 양자 회로입니다.

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

## 에러 처리

### 에러 응답 구조

```json
{
  "error": {
    "code": "error_code",
    "message": "Human-readable error message",
    "details": "Additional details (optional)"
  }
}
```

### 에러 코드

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | `invalid_request` | 잘못된 요청 형식 |
| 400 | `invalid_circuit` | 잘못된 회로 구조 |
| 400 | `qubit_limit_exceeded` | 큐빗 제한 초과 |
| 401 | `unauthorized` | 인증 실패 |
| 401 | `invalid_api_key` | 유효하지 않은 API Key |
| 403 | `forbidden` | 권한 없음 |
| 404 | `not_found` | 리소스를 찾을 수 없음 |
| 404 | `job_not_found` | 작업을 찾을 수 없음 |
| 429 | `rate_limit_exceeded` | 요청 제한 초과 |
| 500 | `internal_error` | 서버 내부 오류 |
| 503 | `queue_full` | 작업 큐 포화 |
| 503 | `compute_unavailable` | 컴퓨트 서버 응답 없음 |

### 에러 예제

**큐빗 제한 초과:**
```json
{
  "error": {
    "code": "invalid_request",
    "message": "n_qubits exceeds your limit of 1,000 qubits (alpha tier)",
    "details": "Requested: 2000, Allowed: 1000"
  }
}
```

**잘못된 게이트:**
```json
{
  "error": {
    "code": "invalid_circuit",
    "message": "Gate 5: target qubit index out of range",
    "details": "Target qubit 10 exceeds circuit size (n_qubits=5)"
  }
}
```

**작업 큐 포화:**
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

## 사용량 조회

### GET `/usage`

API Key별 사용량 통계를 조회합니다.

```http
GET /api/v1/usage?key_id=key_abc123&days=7
Authorization: Bearer YOUR_JWT_TOKEN
```

**Query Parameters:**
- `key_id` (optional): 특정 API Key 필터
- `days` (optional): 조회 기간 (기본: 30일)

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

상세한 요청 로그를 조회합니다.

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

## 시스템 제한

### Alpha 버전 제한

| 항목 | 제한값 | 설명 |
|------|--------|------|
| **큐빗 수** | 1,000 | 사용자당 최대 큐빗 |
| **게이트 수** | 67,108,864 | 회로당 최대 게이트 |
| **측정 샷** | 67,108,864 | 최대 측정 횟수 |
| **동시 작업** | 제한 없음 | 비동기 작업 제출 |
| **요청 크기** | 1,000 MB | HTTP 요청 크기 |

### 시스템 최대 제한

| 항목 | 제한값 | 설명 |
|------|--------|------|
| **최대 큐빗** | 67,108,864 | 시스템 절대 한계 |
| **최대 제어** | 65,536 | Multi-control 게이트 |
| **큐 용량** | 1,024 | 동시 처리 작업 수 |
| **작업 보관** | 1시간 | 완료 후 자동 삭제 |

### 성능 가이드라인

| 회로 크기 | 권장 API | 예상 시간 |
|-----------|----------|-----------|
| < 10 큐빗, < 50 게이트 | 동기 API | < 5초 |
| 10-20 큐빗, < 100 게이트 | 동기/비동기 | 5-30초 |
| 20-100 큐빗 | 비동기 API | 30초-10분 |
| 100-1000 큐빗 | 비동기 API | 10분-2시간 |

---

## 빠른 시작 예제

### Python

```python
import requests
import json

API_KEY = "sk_live_your_api_key_here"
BASE_URL = "https://sethquantum.net/api/v1"

# Bell 상태 회로
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

# 동기 실행
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

// 동기 실행
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

## 지원

- **웹사이트**: https://sethquantum.net
- **대시보드**: https://sethquantum.net/dashboard

---

**Seth Quantum Cloud Gateway API v1.0.0**
*High-Performance Quantum Circuit Simulation as a Service*

© 2025 flyingtext / 윤지현. All rights reserved.
