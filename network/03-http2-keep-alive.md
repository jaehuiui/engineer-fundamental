# HTTP/2 Keep-Alive

HTTP/2에서 "keep alive"가 언급되는 맥락과 HTTP/1.1과의 차이 정리.

## HTTP/1.1의 Keep-Alive

HTTP/1.0에서는 기본이 `Connection: close`였기 때문에 `Connection: keep-alive`로 TCP 연결 재사용을 명시해야 했다. HTTP/1.1부터는 기본이 keep-alive로 바뀌었지만 관행적으로 헤더가 쓰였다.

## HTTP/2는 기본이 Persistent Connection

HTTP/2는 설계 자체가 single persistent connection + multiplexing이다.

```
HTTP/1.1:  요청1 → 응답1 → 요청2 → 응답2  (순차, 또는 여러 커넥션)
HTTP/2:    요청1 ─┐
           요청2 ─┼─ 하나의 TCP 연결에서 동시 처리
           요청3 ─┘
```

- 도메인당 하나의 TCP 연결만 열고, 그 안에서 stream으로 다중화
- `Connection`, `Keep-Alive` 헤더는 HTTP/2 스펙(RFC 9113)에서 명시적으로 금지된 connection-specific 헤더
- 프록시나 중간 장비가 이 헤더를 보면 오히려 오동작할 수 있음

## HTTP/2에서 "Keep Alive"가 언급되는 진짜 맥락

### TCP Keep-Alive

```
# OS 레벨 TCP keepalive — 연결이 죽었는지 감지
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
```

- idle 상태에서 연결이 살아있는지 확인하는 TCP 프로브
- 방화벽/로드밸런서가 idle connection을 끊는 걸 방지

### HTTP/2 PING Frame

```
클라이언트 → PING frame → 서버
서버 → PING ACK → 클라이언트
```

- HTTP/2 자체 프로토콜 레벨의 heartbeat
- 연결 활성 상태 확인 + RTT 측정
- `GOAWAY` frame으로 graceful shutdown도 가능

### gRPC keepalive (HTTP/2 기반)

```js
// gRPC 클라이언트 설정 예시
const client = new grpc.Client(address, credentials, {
  'grpc.keepalive_time_ms': 10000,        // 10초마다 ping
  'grpc.keepalive_timeout_ms': 5000,       // 5초 내 응답 없으면 dead
  'grpc.keepalive_permit_without_calls': 1 // 요청 없어도 ping
});
```

## 정리

| 레벨 | 목적 | HTTP/2에서 |
|------|------|-----------|
| HTTP/1.1 `Keep-Alive` 헤더 | TCP 연결 재사용 | 사용 금지 (불필요) |
| TCP keepalive | idle 연결 생존 확인 | OS 레벨에서 여전히 유효 |
| HTTP/2 PING frame | 프로토콜 레벨 heartbeat | HTTP/2 고유 기능 |
| gRPC keepalive | 장기 연결 관리 | HTTP/2 PING 기반 |

HTTP/2에서 "keep alive"를 쓴다고 하면, HTTP/1.1의 `Keep-Alive` 헤더가 아니라 TCP keepalive나 HTTP/2 PING frame을 말하는 것. 연결 자체는 기본으로 살아있지만, idle 상태에서 중간 장비(LB, 방화벽)가 끊지 않도록 주기적으로 heartbeat를 보내는 용도.
