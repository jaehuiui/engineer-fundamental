# HTTP Protocol Deep Dive

HTTP 프로토콜의 버전 협상 (Version Negotiation), Keep-Alive, 그리고 버전별 심화 동작 정리.

## HTTP Version Negotiation (HTTP 버전 협상)

브라우저가 서버와 통신할 때, 어떤 HTTP 버전을 사용할지 어떻게 결정하는가?

### HTTPS 환경 — ALPN (Application-Layer Protocol Negotiation)

HTTP/2는 사실상 HTTPS에서만 동작한다. 버전 결정은 TLS Handshake 안에서 일어난다.

```
1. TCP 3-Way Handshake (연결 설정, Connection Establishment)
   SYN → SYN-ACK → ACK

2. TLS Handshake (암호화 + HTTP 버전 협상)
   Client Hello에 ALPN 확장 (extension) 포함:
     "나는 h2, http/1.1 을 지원해"

   Server Hello에서 선택:
     "h2로 하자" (또는 "http/1.1로 하자")

3. HTTP 통신 시작 (합의된 버전으로)
```

핵심은 ALPN이다. TLS Handshake의 일부로, 추가 왕복 (Round Trip) 없이 HTTP 버전을 결정한다.

```
Client Hello:
  - TLS version: 1.3
  - Cipher suites: [...]
  - ALPN: ["h2", "http/1.1"]     ← "나 이거 지원해 (I support these)"
  - SNI: example.com

Server Hello:
  - Selected cipher: AES_256_GCM
  - ALPN: "h2"                    ← "그럼 h2로 하자 (Let's use h2)"
  - Certificate: [...]
```

서버가 h2를 지원하지 않으면 http/1.1로 폴백 (fallback)한다.

### HTTP 평문 (Cleartext) 환경 — Upgrade 헤더

TLS가 없으면 ALPN을 쓸 수 없다. 이 경우 HTTP/1.1의 Upgrade 메커니즘을 사용한다.

```
# 클라이언트가 HTTP/1.1로 요청하면서 업그레이드 제안 (Upgrade Proposal)
GET / HTTP/1.1
Host: example.com
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c                        ← "HTTP/2 cleartext로 업그레이드 할래?"
HTTP2-Settings: <base64 settings>

# 서버가 수락하면 (Server Accepts)
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: h2c

# 이후 HTTP/2로 통신
```

하지만 실제로 브라우저들은 평문 HTTP/2(h2c)를 지원하지 않는다.
브라우저에서 HTTP/2는 HTTPS 전용. h2c는 서버 간 통신 (gRPC 등)에서 사용.

### HTTP/3 (QUIC) — Alt-Svc 기반 발견 (Discovery-Based)

HTTP/3는 TCP가 아닌 UDP(QUIC) 기반이라 완전히 다른 방식으로 협상한다.

```
1. 첫 방문 (First Visit): 브라우저는 HTTP/3 지원 여부를 모름
   → 일반적인 TCP + TLS + HTTP/2로 연결

2. 서버가 Alt-Svc 헤더로 HTTP/3 지원을 알림 (Advertise)
   HTTP/2 200 OK
   Alt-Svc: h3=":443"; ma=86400    ← "나 UDP 443에서 HTTP/3도 돼, 24시간 유효"

3. 다음 요청부터 브라우저가 QUIC(UDP)으로 연결 시도
   → 성공하면 HTTP/3
   → 실패하면 기존 HTTP/2 유지 (fallback)
```

HTTP/3는 "발견 (Discovery)" 기반이다. 처음부터 HTTP/3로 연결하는 게 아니라,
HTTP/2 응답에서 Alt-Svc를 보고 다음부터 시도한다.

### 전체 흐름 (Full Negotiation Flow)

```
브라우저가 https://example.com 요청:

[첫 방문]
  TCP Handshake → TLS Handshake (ALPN: h2, http/1.1)
    → 서버가 h2 선택 → HTTP/2로 통신
    → 응답에 Alt-Svc: h3=":443" 포함

[다음 방문]
  QUIC Handshake (UDP, 0-RTT 가능)
    → 성공: HTTP/3로 통신
    → 실패: TCP + TLS fallback → HTTP/2
```

### 버전별 협상 방식 비교 (Negotiation Comparison)

| 전환 (Transition) | 협상 방식 (Method) | 추가 비용 (Extra Round Trip) |
|------|----------|----------------------|
| HTTP/1.1 → HTTP/2 | TLS ALPN | 0 (TLS Handshake에 포함) |
| HTTP/1.1 → HTTP/2 (평문) | Upgrade 헤더 | 1 RT (101 응답 대기) |
| HTTP/2 → HTTP/3 | Alt-Svc 헤더 | 0 (다음 요청부터 적용) |

## TLS — Transport Layer Security

HTTPS의 "S"가 바로 TLS다. TCP 연결 위에서 데이터를 암호화하는 프로토콜 (Encryption Protocol over TCP).

### TLS가 하는 일 (What TLS Does)

```
1. 인증 (Authentication): 서버가 진짜 example.com인지 인증서로 확인
2. 기밀성 (Confidentiality): 데이터를 암호화하여 도청 방지 (Eavesdropping Prevention)
3. 무결성 (Integrity): 데이터가 중간에 변조되지 않았음을 보장 (Tamper Prevention)
```

### TLS 1.3 Handshake 과정 (Handshake Flow)

```
Client                              Server
  │                                    │
  │──── Client Hello ────────────────→│
  │     - 지원하는 cipher suites       │
  │     - Client Random (난수)         │
  │     - ALPN: ["h2", "http/1.1"]    │
  │     - SNI: example.com            │
  │     - Key Share (DH 공개키)        │
  │                                    │
  │←─── Server Hello ────────────────│
  │     - 선택된 cipher suite          │
  │     - Server Random               │
  │     - Key Share (DH 공개키)        │
  │     - Certificate (인증서)         │
  │     - Certificate Verify (서명)    │
  │     - Finished                     │
  │                                    │
  │──── Finished ────────────────────→│
  │                                    │
  │     [암호화된 HTTP 통신 시작]        │
```

TLS 1.3은 1-RTT (한 번의 왕복)로 핸드셰이크가 완료된다. TLS 1.2는 2-RTT가 필요했다.

### 주요 개념 (Key Concepts)

**인증서 체인 (Certificate Chain):**
```
Root CA (브라우저에 내장, Trusted Root)
  └─ Intermediate CA (중간 인증 기관)
       └─ example.com 인증서 (서버가 제공)

브라우저는 이 체인을 따라 올라가며 신뢰할 수 있는 Root CA에 도달하는지 확인.
도달하면 → 신뢰 (Trusted)
도달 못하면 → "이 연결은 안전하지 않습니다" 경고
```

**SNI (Server Name Indication):**
```
하나의 IP에 여러 도메인이 있을 때, 어떤 도메인의 인증서를 보내야 하는지 알려주는 확장.

Client Hello:
  SNI: example.com    ← "나는 example.com에 접속하려는 거야"

→ 서버가 example.com의 인증서를 선택해서 보냄
→ 주의: SNI는 암호화되지 않음 (평문, Plaintext) → 어떤 사이트에 접속하는지 노출
   → ECH (Encrypted Client Hello)로 해결 시도 중
```

**0-RTT Resumption (세션 재개):**
```
[첫 연결] 1-RTT Handshake → 세션 티켓 (Session Ticket) 저장

[재연결] 0-RTT:
  Client Hello + 이전 세션 티켓 + 암호화된 HTTP 요청 (동시에!)
  → 서버가 티켓 검증 후 즉시 응답
  → Handshake 완료 전에 데이터 전송 시작 (Early Data)

주의: 0-RTT는 재전송 공격 (Replay Attack)에 취약
  → GET 같은 멱등 (Idempotent) 요청에만 사용해야 안전
```

**Forward Secrecy (전방 비밀성):**
```
서버의 개인키가 유출되어도, 과거 통신 내용을 복호화할 수 없음.
→ 매 세션마다 새로운 임시 키 (Ephemeral Key)를 생성하기 때문
→ TLS 1.3에서는 강제 (Mandatory)
```

### TLS 버전 비교 (Version Comparison)

| | TLS 1.2 | TLS 1.3 |
|---|---|---|
| Handshake RTT | 2-RTT | 1-RTT (재연결 시 0-RTT) |
| Cipher Suites | 많음 (취약한 것 포함) | 5개만 (안전한 것만) |
| 키 교환 (Key Exchange) | RSA 또는 DH | DH만 (Forward Secrecy 강제) |
| 암호화 시작 시점 | Handshake 완료 후 | Server Hello 직후 |

## Keep-Alive — 버전별 연결 유지 (Connection Persistence)

### HTTP/1.0 — 매 요청마다 새 연결 (Connection per Request)

```
요청1: TCP 연결 → HTTP 요청 → HTTP 응답 → TCP 종료
요청2: TCP 연결 → HTTP 요청 → HTTP 응답 → TCP 종료
→ 매번 3-Way Handshake 비용 발생
```

### HTTP/1.1 — Keep-Alive 기본 (Persistent Connection by Default)

```
TCP 연결 → 요청1 → 응답1 → 요청2 → 응답2 → ... → TCP 종료
→ 하나의 연결로 여러 요청 처리 (Connection Reuse)
```

HTTP/1.0에서는 `Connection: keep-alive`를 명시해야 했지만,
HTTP/1.1부터는 기본이 keep-alive. `Connection: close`를 보내야 연결이 끊긴다.

### HTTP/2 — 단일 영속 연결 + 다중화 (Single Persistent Connection + Multiplexing)

```
HTTP/1.1:  요청1 → 응답1 → 요청2 → 응답2  (순차, Head-of-Line Blocking)
HTTP/2:    요청1 ─┐
           요청2 ─┼─ 하나의 TCP 연결에서 동시 처리 (Multiplexing)
           요청3 ─┘
```

- 도메인당 하나의 TCP 연결만 열고, 그 안에서 stream으로 다중화 (Multiplexing)
- `Connection`, `Keep-Alive` 헤더는 HTTP/2 스펙 (RFC 9113)에서 명시적으로 금지 (Prohibited)
- 프록시나 중간 장비가 이 헤더를 보면 오히려 오동작할 수 있음

### HTTP/2에서 "Keep Alive"가 언급되는 진짜 맥락 (Actual Keep-Alive in HTTP/2)

**TCP Keep-Alive — OS 레벨 연결 생존 확인 (OS-Level Connection Probe):**
```
# OS 레벨 TCP keepalive
net.ipv4.tcp_keepalive_time = 300     # 300초 idle 후 프로브 시작
net.ipv4.tcp_keepalive_intvl = 75     # 75초 간격으로 프로브
net.ipv4.tcp_keepalive_probes = 9     # 9번 실패하면 연결 종료
```
- idle 상태에서 연결이 살아있는지 확인하는 TCP 프로브 (Probe)
- 방화벽/로드밸런서가 idle connection을 끊는 걸 방지 (Prevent Idle Timeout)

**HTTP/2 PING Frame — 프로토콜 레벨 하트비트 (Protocol-Level Heartbeat):**
```
클라이언트 → PING frame → 서버
서버 → PING ACK → 클라이언트
```
- HTTP/2 자체 프로토콜 레벨의 heartbeat
- 연결 활성 상태 확인 (Liveness Check) + RTT 측정
- `GOAWAY` frame으로 우아한 종료 (Graceful Shutdown)도 가능

**gRPC keepalive (HTTP/2 기반):**
```js
const client = new grpc.Client(address, credentials, {
  'grpc.keepalive_time_ms': 10000,        // 10초마다 ping
  'grpc.keepalive_timeout_ms': 5000,       // 5초 내 응답 없으면 dead
  'grpc.keepalive_permit_without_calls': 1 // 요청 없어도 ping
});
```

### Keep-Alive 정리 (Summary)

| 레벨 (Level) | 목적 (Purpose) | HTTP/2에서 (In HTTP/2) |
|------|------|-----------|
| HTTP/1.1 `Keep-Alive` 헤더 | TCP 연결 재사용 (Connection Reuse) | 사용 금지 (Prohibited) |
| TCP keepalive | idle 연결 생존 확인 (Idle Connection Probe) | OS 레벨에서 여전히 유효 |
| HTTP/2 PING frame | 프로토콜 레벨 하트비트 (Heartbeat) | HTTP/2 고유 기능 |
| gRPC keepalive | 장기 연결 관리 (Long-Lived Connection) | HTTP/2 PING 기반 |

## 브라우저에서 UDP 통신 (UDP in Browsers)

### 브라우저는 순수 UDP 소켓을 열 수 없다 (Browsers Cannot Open Raw UDP Sockets)

```
브라우저 = 샌드박스 환경 (Sandboxed Environment)
  → 임의의 네트워크 소켓을 열 수 없음
  → TCP 소켓도 직접 못 열고, HTTP/WebSocket API를 통해서만 통신
  → UDP는 더더욱 불가능

왜? (Why?)
  1. 보안 (Security): 악성 웹사이트가 UDP로 DNS 스푸핑, DDoS 공격 가능
  2. 동일 출처 정책 (Same-Origin Policy): UDP에는 Origin 개념이 없음
  3. 포트 스캔 방지 (Port Scan Prevention): 내부 네트워크 탐색 가능
```

브라우저의 네트워크 API는 전부 HTTP 기반이거나, 브라우저가 중간에서 제어하는 형태:
```
fetch() / XMLHttpRequest  → HTTP(S) 전용 (TCP 기반)
WebSocket                 → HTTP Upgrade 후 TCP 기반
WebRTC                    → 브라우저가 관리하는 UDP (직접 제어 불가)
```

### HTTP/3에서 브라우저가 UDP를 쓰는 방식 (How Browsers Use UDP for HTTP/3)

```
개발자가 하는 일: fetch('https://example.com')  ← 그냥 평범한 fetch
브라우저가 하는 일: QUIC(UDP) 연결을 내부적으로 처리

개발자는 TCP인지 UDP인지 알 수도 없고, 제어할 수도 없다.
JavaScript에서 "UDP로 보내줘"라고 지정하는 API는 없다.
```

브라우저가 HTTP/3를 쓸지 말지는 전적으로 브라우저의 판단 (Browser's Decision):
```
1. 서버가 Alt-Svc: h3=":443" 헤더를 보냄
2. 브라우저가 "다음에 QUIC으로 시도해볼까" 판단
3. QUIC 연결 성공 → UDP 사용 (개발자 모름)
4. QUIC 연결 실패 → TCP fallback (개발자 모름)

→ 개발자가 "이 요청은 UDP로 보내"라고 강제할 방법 없음
```

### WebRTC — 브라우저가 관리하는 UDP (Browser-Managed UDP)

WebRTC는 브라우저에서 UDP 통신이 실제로 일어나는 유일한 개발자 접근 가능 경로.
하지만 "순수 UDP"가 아니라 브라우저가 감싸서 (Wrap) 제공하는 형태.

```
순수 UDP:
  socket = new UDPSocket()     ← 이런 API 없음! (Does NOT exist)
  socket.send(data, host, port)

WebRTC (브라우저가 관리하는 UDP):
  RTCPeerConnection → ICE/STUN/TURN 협상 → DTLS 암호화 → SRTP/SCTP
  → 내부적으로 UDP를 사용하지만, 개발자가 UDP 패킷을 직접 만들 수 없음
```

```js
// WebRTC Data Channel — UDP에 가장 가까운 브라우저 API
const pc = new RTCPeerConnection();
const dc = pc.createDataChannel('game', {
  ordered: false,      // 순서 보장 안 함 (No ordering — UDP-like)
  maxRetransmits: 0    // 재전송 안 함 (No retransmission — fire and forget)
});
// → 내부적으로 UDP를 사용하지만, 개발자가 보는 건 DataChannel API
```

### WebTransport — UDP에 가장 가까운 미래 API (Closest to Raw UDP)

```js
// WebTransport = HTTP/3(QUIC) 위에서 동작하는 양방향 통신 API
const transport = new WebTransport('https://example.com:4433');
await transport.ready;

// 비신뢰성 데이터그램 (Unreliable Datagram) — UDP와 가장 유사
const writer = transport.datagrams.writable.getWriter();
await writer.write(new Uint8Array([1, 2, 3]));
// → 순서 보장 X, 도착 보장 X, 재전송 X — UDP처럼 동작

// 신뢰성 스트림 (Reliable Stream) — TCP처럼 동작
const stream = await transport.createBidirectionalStream();
```

### 브라우저 통신 API 비교 (Browser Communication API Comparison)

| | WebSocket | WebRTC DataChannel | WebTransport |
|---|---|---|---|
| 프로토콜 (Protocol) | TCP | UDP (SCTP/DTLS) | UDP (QUIC) |
| 서버 필요 (Server) | WebSocket 서버 | STUN/TURN + 시그널링 | HTTP/3 서버 |
| 비신뢰성 전송 (Unreliable) | X | O | O |
| 다중 스트림 (Multiplexing) | X | O | O |
| 설정 복잡도 (Complexity) | 낮음 (Low) | 높음 (High) | 중간 (Medium) |
| 용도 (Use Case) | 채팅, 알림 | P2P, 화상통화 | 게임, 실시간 스트리밍 |

### 실제 브라우저 앱에서 UDP가 사용되는 방식 (How Real Browser Apps Use UDP)

브라우저 위에서 게임, 스트리밍, VoIP를 실행할 때도 항상 래핑된 UDP (Wrapped UDP)를 사용한다.

```
네이티브 앱 (Native App):
  앱 → OS UDP 소켓 → 네트워크
  → 순수 UDP 직접 사용 가능 (Raw UDP)

브라우저 앱 (Browser App):
  앱 → 브라우저 API → 브라우저가 관리하는 UDP → 네트워크
  → 순수 UDP 불가, 반드시 브라우저 API를 통해 래핑 (Always Wrapped)
```

**브라우저 게임 (Browser Games):**
```
방법 1: WebSocket (TCP 기반 — 대부분의 브라우저 게임)
  → 간단하지만 TCP라서 지연 발생 가능 (Head-of-Line Blocking)
  → 턴제 게임, 캐주얼 게임에 적합

방법 2: WebRTC DataChannel (UDP 기반 — 실시간 멀티플레이어)
  → 내부: SCTP over DTLS over UDP
  → ordered: false, maxRetransmits: 0 으로 UDP처럼 동작

방법 3: WebTransport (UDP 기반 — 차세대)
  → 내부: QUIC over UDP
  → 비신뢰성 데이터그램 (Unreliable Datagram) 지원
```

**브라우저 화상통화 (Browser Video Call — Google Meet, Zoom Web):**
```
WebRTC 사용:
  navigator.mediaDevices.getUserMedia({ video: true, audio: true })
    → MediaStream → RTCPeerConnection에 트랙 추가
    → 내부적으로 SRTP over UDP로 전송

프로토콜 스택 (Protocol Stack):
  [영상/음성 데이터]
    → SRTP (Secure Real-time Transport Protocol) — 미디어 암호화
      → DTLS (Datagram TLS) — 키 교환, 암호화 설정
        → ICE (NAT 통과, NAT Traversal)
          → UDP

개발자가 UDP 패킷을 직접 만드는가? → 아니다 (No).
브라우저가 모든 UDP 관련 처리를 내부적으로 수행.
```

**브라우저 라이브 스트리밍 (Browser Live Streaming):**
```
시청자 (Viewer):
  대부분 HLS/DASH → HTTP(TCP) 기반 → 청크 단위 다운로드
  → UDP 아님, 적응형 비트레이트 (Adaptive Bitrate)로 품질 조절

송출자 (Broadcaster):
  WebRTC → UDP 기반 → 초저지연 (Sub-second Latency, ~100ms)

초저지연 필요: WebRTC / WebTransport (UDP) — ~100ms
일반 스트리밍: HLS/DASH (TCP) — 2~30초 지연, 하지만 안정적
```

### 래핑 계층 비교 (Wrapping Layer Comparison)

```
순수 UDP (네이티브 앱):
  [앱 데이터] → UDP → IP
  → 암호화 없음, 인증 없음

WebRTC (브라우저):
  [앱 데이터] → SCTP → DTLS → ICE → UDP → IP
  → 암호화 (DTLS), NAT 통과 (ICE), 흐름 제어 (SCTP)

WebTransport (브라우저):
  [앱 데이터] → QUIC → UDP → IP
  → 암호화 (TLS 1.3), 혼잡 제어 (Congestion Control), 다중 스트림

HTTP/3 (브라우저, 자동):
  [HTTP 요청] → QUIC → UDP → IP
  → 개발자 제어 불가, 브라우저가 자동 처리
```

| | 순수 UDP (Raw) | WebRTC | WebTransport | HTTP/3 |
|---|---|---|---|---|
| 환경 (Env) | 네이티브 앱 | 브라우저 | 브라우저 | 브라우저 |
| 암호화 (Encryption) | 없음 (선택) | DTLS (강제) | TLS 1.3 (강제) | TLS 1.3 (강제) |
| NAT 통과 | 직접 구현 | ICE (자동) | 서버 직접 연결 | 서버 직접 연결 |
| 비신뢰성 (Unreliable) | 기본 | DataChannel 옵션 | Datagram API | X |
| 개발자 제어 (Control) | 완전 (Full) | API 통해 (Via API) | API 통해 | 없음 (None) |
