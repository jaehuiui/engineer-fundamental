# Network Fundamentals — From IP to HTTP

네트워크의 탄생부터 HTTP까지의 발전 과정 정리.

## 1. 시작 — 패킷 교환의 탄생 (1960s)

냉전 시대, 미국 국방부(DoD)는 핵 공격에도 살아남는 통신 네트워크가 필요했다.

**Circuit Switching (회선 교환) — 기존 전화망:**
```
A ──── 교환기 ──── B
         │
       (파괴되면 통신 불가)
```

**Packet Switching (패킷 교환) — 새로운 아이디어:**
```
A ─┬─ 노드1 ─┬─ 노드3 ─── B
   └─ 노드2 ─┘
   (한 경로가 죽어도 다른 경로로 전달)
```

데이터를 작은 패킷으로 쪼개서 여러 경로로 보내고, 도착지에서 재조립하는 방식.

## 2. ARPANET (1969)

미국 DARPA가 만든 최초의 패킷 교환 네트워크. UCLA, 스탠포드, UCSB, 유타 대학 4개 노드로 시작.

```
UCLA ──── SRI (스탠포드)
  │            │
UCSB ──── Utah
```

문제: 각 컴퓨터마다 통신 방식이 달랐다. 통신 규칙에 대한 합의가 필요했다.

## 3. IP — Internet Protocol (1974~1981)

네트워크에 연결된 각 장치에 고유한 주소를 부여하고, 패킷을 목적지까지 전달하는 프로토콜.

```
IP 패킷 구조:
┌──────────────────────────────────┐
│ Header                           │
│  - Source IP:      192.168.1.1   │  ← 보내는 사람 주소
│  - Destination IP: 10.0.0.1      │  ← 받는 사람 주소
│  - TTL: 64                       │  ← 최대 홉 수 (무한 루프 방지)
│  - Protocol: TCP(6) / UDP(17)    │
├──────────────────────────────────┤
│ Payload (실제 데이터)               │
└──────────────────────────────────┘
```

**IP의 특성:**
- Connectionless — 연결 설정 없이 패킷을 보냄
- Best-effort — 도착을 보장하지 않음 (패킷이 유실될 수 있음)
- 각 패킷이 독립적으로 라우팅됨

**IP 주소 체계:**
```
IPv4: 32비트, 약 43억 개
  192.168.1.1 → 11000000.10101000.00000001.00000001

IPv6: 128비트, 사실상 무한
  2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

**라우팅 — 패킷이 목적지를 찾아가는 방법:**
```
A (192.168.1.1) → Router1 → Router2 → Router3 → B (10.0.0.1)

각 라우터는 라우팅 테이블을 보고 "다음 홉"을 결정:
  목적지 10.0.0.0/24 → Router2로 전달
  목적지 172.16.0.0/16 → Router5로 전달
```

IP만으로는 부족했다. 패킷이 유실되거나 순서가 뒤바뀌어도 IP는 신경 쓰지 않는다.

## 4. TCP — Transmission Control Protocol

IP 위에서 신뢰성 있는 통신을 보장하는 프로토콜.

**TCP가 해결하는 문제:**
- 패킷 유실 → 재전송
- 순서 뒤바뀜 → 시퀀스 번호로 재정렬
- 중복 패킷 → 감지 후 폐기
- 흐름 제어 → 수신자가 처리할 수 있는 속도로 조절
- 혼잡 제어 → 네트워크 과부하 방지

**3-Way Handshake (연결 설정):**

TCP에서 처음 도입된 개념 (1974년 Vint Cerf & Bob Kahn 논문, 1981년 RFC 793 표준화).

```
Client                    Server
  │                         │
  │──── SYN (seq=100) ────→│   1. "연결하고 싶어"
  │                         │
  │←── SYN-ACK ───────────│   2. "좋아, 나도 준비됐어"
  │    (seq=300, ack=101)   │
  │                         │
  │──── ACK (ack=301) ────→│   3. "확인, 시작하자"
  │                         │
  │    [데이터 전송 시작]     │
```

**왜 2-Way가 아닌 3-Way인가?**
```
2-Way의 문제:
  Client → SYN (오래된 패킷, 지연 도착)
  Server ← SYN-ACK ("연결 수락!")
  Server: 연결 열고 대기... 하지만 Client는 이 연결을 모름
  → 리소스 낭비, 유령 연결 (half-open connection)

3-Way로 해결:
  Client → SYN
  Server ← SYN-ACK
  Client → ACK  ← Client가 "맞아, 내가 요청한 거야"를 확인
```
3번째 ACK가 있어야 양쪽 모두 "상대방이 살아있고, 내 시퀀스 번호를 확인했다"는 걸 보장할 수 있다.

**TCP 세그먼트 구조:**
```
┌─────────────────────────────────┐
│ Source Port: 54321               │  ← 보내는 프로세스
│ Destination Port: 80             │  ← 받는 프로세스 (HTTP)
│ Sequence Number: 1001            │  ← 이 데이터의 순서
│ Acknowledgment Number: 5001      │  ← 상대방 데이터 확인
│ Flags: SYN, ACK, FIN, RST ...   │
│ Window Size: 65535               │  ← 흐름 제어
├─────────────────────────────────┤
│ Payload                          │
└─────────────────────────────────┘
```

**Port의 등장:**

IP는 "어떤 컴퓨터"까지만 식별한다. 하나의 컴퓨터에서 여러 프로그램이 동시에 통신하려면? → Port 번호로 프로세스를 구분.

```
IP = 아파트 건물 주소
Port = 호수

192.168.1.1:80   → 이 컴퓨터의 웹 서버
192.168.1.1:443  → 이 컴퓨터의 HTTPS 서버
192.168.1.1:3306 → 이 컴퓨터의 MySQL
```

**4-Way Handshake (연결 종료):**
```
Client                    Server
  │──── FIN ─────────────→│   1. "나 보낼 거 다 보냈어"
  │←──── ACK ─────────────│   2. "알겠어"
  │←──── FIN ─────────────│   3. "나도 다 보냈어"
  │──── ACK ─────────────→│   4. "확인, 끝"
```

## 5. UDP — User Datagram Protocol

TCP의 신뢰성이 필요 없거나, 속도가 더 중요한 경우를 위한 프로토콜.

```
TCP:  연결 설정 → 데이터 전송 (순서 보장, 재전송) → 연결 종료
UDP:  그냥 보냄. 끝.
```

**UDP 데이터그램 구조:**
```
┌─────────────────────────┐
│ Source Port: 54321       │
│ Destination Port: 53    │  ← DNS
│ Length: 512              │
│ Checksum                 │
├─────────────────────────┤
│ Payload                  │
└─────────────────────────┘
```

TCP 대비 헤더가 매우 작다 (8바이트 vs TCP 20바이트+).

**TCP vs UDP:**

| | TCP | UDP |
|---|---|---|
| 연결 | 연결 지향 (handshake) | 비연결 |
| 신뢰성 | 보장 (재전송, 순서) | 보장 안 함 |
| 속도 | 상대적으로 느림 | 빠름 |
| 헤더 크기 | 20~60 바이트 | 8 바이트 |
| 용도 | HTTP, 파일 전송, 이메일 | DNS, 스트리밍, 게임, VoIP |

**왜 UDP를 쓰는가?**
- DNS 조회: 작은 요청/응답, 빠른 속도 필요
- 실시간 스트리밍: 패킷 하나 유실돼도 다음 프레임 보여주면 됨
- 온라인 게임: 0.1초 전 위치 데이터를 재전송받는 것보다 최신 데이터가 중요
- QUIC (HTTP/3): UDP 위에 자체 신뢰성 구현

### TCP와 UDP는 동시에 존재할 수 있는가?

그렇다. TCP와 UDP는 독립적인 프로토콜이며, 같은 장치에서 동시에 사용 가능하다.

```
같은 서버에서:
  TCP 80   → HTTP/1.1 웹 서버
  TCP 443  → HTTPS (HTTP/2)
  UDP 443  → QUIC (HTTP/3)
  UDP 53   → DNS 서버
  TCP 3306 → MySQL

같은 포트 번호라도 TCP와 UDP는 별개:
  TCP 443 ≠ UDP 443  → 서로 다른 소켓, 다른 프로토콜
```

실제로 HTTP/3를 지원하는 서버는 TCP 443(HTTP/2)과 UDP 443(QUIC)을 동시에 열어둔다.

### 소켓 — 네트워크 통신의 기본 단위

프로그램이 네트워크 통신을 하려면 OS에 "소켓"을 요청한다. 소켓을 만들 때 TCP인지 UDP인지 지정한다.

```js
// Node.js — 같은 서버에서 TCP와 UDP를 동시에 열기

// TCP 서버 (HTTP)
const http = require('http');
const httpServer = http.createServer((req, res) => {
  res.end('Hello from TCP');
});
httpServer.listen(443);  // TCP 443 포트 열기

// UDP 서버 (같은 포트 번호도 가능)
const dgram = require('dgram');
const udpServer = dgram.createSocket('udp4');
udpServer.bind(443);     // UDP 443 포트 열기 — TCP 443과 별개!
```

**OS 레벨 시스템 콜:**
```
TCP:
  socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)  → TCP 소켓 생성
  bind(fd, addr:443)                          → 443 포트에 바인딩
  listen(fd, backlog)                         → 연결 대기 시작
  accept(fd)                                  → 클라이언트 연결 수락

UDP:
  socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)   → UDP 소켓 생성
  bind(fd, addr:443)                          → 443 포트에 바인딩
  recvfrom(fd)                                → 데이터그램 수신 대기
```

**OS 커널 내부 — TCP와 UDP는 별개의 테이블로 관리:**
```
TCP 포트 테이블:
  80  → nginx (PID 1234)
  443 → nginx (PID 1234)
  3306 → mysqld (PID 5678)

UDP 포트 테이블:
  53  → named (PID 9012)
  443 → nginx (PID 1234)    ← TCP 443과 별개!
  514 → syslogd (PID 3456)
```

### 실제 사례 — DNS (TCP + UDP 동시 사용)

```
DNS 서버 (포트 53):
  UDP 53 → 일반 DNS 조회 (512바이트 이하, 빠른 응답)
  TCP 53 → 큰 응답 (512바이트 초과), 존 전송 (AXFR)

클라이언트가 UDP로 질의 → 응답이 크면 TC(Truncated) 플래그 설정
→ 클라이언트가 TCP로 재질의
```

### 실제 사례 — nginx HTTP/2 + HTTP/3 동시 지원

```nginx
server {
    listen 443 ssl http2;           # TCP 443 — HTTP/1.1, HTTP/2
    listen 443 quic reuseport;      # UDP 443 — HTTP/3 (QUIC)

    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    add_header Alt-Svc 'h3=":443"; ma=86400';  # HTTP/3 지원 알림
}
```

### 소켓의 생명주기 — 열기/닫기/제어

**TCP 소켓:**
```
socket() → bind() → listen() → accept() → read/write → close()
                                  ↑                        │
                                  └──── 새 클라이언트 ──────┘
```

**UDP 소켓:**
```
socket() → bind() → recvfrom/sendto → close()
                        ↑                 │
                        └──── 반복 ───────┘
```

**열린 포트 확인 및 제어:**
```bash
# 현재 열린 포트 확인
netstat -tlnp    # TCP 리스닝 포트
netstat -ulnp    # UDP 리스닝 포트
ss -tlnp         # 더 빠른 대안 (Linux)
lsof -i :443     # 443 포트 사용 중인 프로세스

# 결과 예시:
Proto  Local Address    State       PID/Program
tcp    0.0.0.0:443     LISTEN      1234/nginx
udp    0.0.0.0:443                 1234/nginx
tcp    0.0.0.0:80      LISTEN      1234/nginx
```

**프로그래밍으로 제어:**
```js
// 서버 시작 (포트 열기)
server.listen(443, () => console.log('TCP 443 열림'));

// 서버 종료 (포트 닫기 — 새 연결 거부, 기존 연결은 유지)
server.close(() => console.log('TCP 443 닫힘'));
```

**방화벽으로 제어:**
```bash
# iptables (Linux) — TCP/UDP 별도 제어
iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # TCP 443 허용
iptables -A INPUT -p udp --dport 443 -j ACCEPT   # UDP 443 허용
iptables -A INPUT -p tcp --dport 3306 -j DROP     # TCP 3306 차단
```

### HTTP와 Transport Protocol의 관계

```
HTTP/0.9 ~ HTTP/2  →  TCP 기반
HTTP/3              →  UDP 기반 (QUIC)

HTTP/3 (Application)
  └─ QUIC (Transport, UDP 위에 구현)
       └─ UDP (Transport)
            └─ IP (Network)
```

QUIC는 UDP 위에서 TCP의 장점(신뢰성, 순서 보장, 혼잡 제어)을 자체 구현한 프로토콜이다.
UDP 자체는 신뢰성이 없지만, QUIC가 그 위에서 신뢰성을 보장한다.

### 브라우저에서 UDP 기반 통신은 어떻게 동작하는가?

브라우저는 직접 UDP 소켓을 열 수 없다 (보안상 차단). 하지만 다음 방식으로 UDP 기반 통신이 동작한다:

```
1. HTTP/3 (QUIC)
   브라우저가 자동으로 처리. 개발자가 신경 쓸 필요 없음.
   서버가 Alt-Svc 헤더로 HTTP/3 지원을 알리면 브라우저가 자동 업그레이드.

   HTTP/1.1 200 OK
   Alt-Svc: h3=":443"; ma=86400    ← "나 HTTP/3도 돼"
   → 다음 요청부터 브라우저가 QUIC(UDP)로 전환

2. WebRTC
   브라우저 간 P2P 통신. 내부적으로 UDP 사용.
   → 화상통화, 화면 공유, P2P 파일 전송
   → ICE/STUN/TURN으로 NAT 통과

3. DNS over HTTPS (DoH)
   DNS 조회를 HTTPS로 감싸서 보냄 (실제로는 TCP).
   하지만 전통적 DNS는 UDP 53 포트 사용.
   브라우저가 OS DNS 대신 DoH를 직접 사용하기도 함.
```

| 방식 | 프로토콜 | 브라우저 API | 개발자 제어 |
|------|----------|-------------|------------|
| HTTP/3 | QUIC (UDP) | fetch/XHR (자동) | 불필요 (브라우저 자동) |
| WebRTC | SRTP/SCTP (UDP) | RTCPeerConnection | 직접 제어 |
| WebSocket | TCP | WebSocket API | 직접 제어 |
| 일반 HTTP | TCP | fetch/XHR | 직접 제어 |

## 6. OSI 7 Layer & TCP/IP 4 Layer

이 모든 프로토콜을 계층으로 정리한 모델:

```
OSI 7 Layer          TCP/IP 4 Layer       프로토콜 예시
─────────────────────────────────────────────────────
7. Application  ─┐
6. Presentation  ├→ Application          HTTP, DNS, FTP, SMTP
5. Session      ─┘
4. Transport    ───→ Transport           TCP, UDP
3. Network      ───→ Internet            IP, ICMP, ARP
2. Data Link    ─┐
1. Physical     ─┴→ Network Access       Ethernet, Wi-Fi
```

각 계층은 아래 계층의 서비스를 사용하고, 위 계층에 서비스를 제공한다:
```
HTTP 메시지
  → TCP 세그먼트로 감싸짐 (포트 번호 추가)
    → IP 패킷으로 감싸짐 (IP 주소 추가)
      → Ethernet 프레임으로 감싸짐 (MAC 주소 추가)
        → 전기 신호 / 광 신호로 전송
```

## 7. DNS — Domain Name System (1983)

IP 주소를 외우기 어려우니, 사람이 읽을 수 있는 이름을 매핑하는 시스템.

### DNS 조회 과정

브라우저에 `https://example.com`을 입력하면:

**Step 1: 로컬 캐시 확인**
```
브라우저 DNS 캐시 → OS DNS 캐시 (nscd/systemd-resolved)
  → hosts 파일 (/etc/hosts)
  → 공유기 DNS 캐시
```
여기서 찾으면 네트워크 요청 없이 즉시 반환.

**Step 2: Recursive Resolver (재귀 DNS 서버)**

ISP가 제공하거나, 직접 설정한 DNS 서버 (예: `8.8.8.8` Google, `1.1.1.1` Cloudflare).
이 서버가 "대신" 여러 DNS 서버를 돌아다니며 답을 찾아준다.

**Step 3: Root DNS Server**

전 세계에 13개 루트 서버 그룹 (a~m.root-servers.net). 실제로는 Anycast로 수백 대가 분산 운영.
```
질문: "example.com의 IP는?"
루트 서버: "나는 모르지만, .com을 담당하는 TLD 서버 주소를 알려줄게"
         → a.gtld-servers.net (192.5.6.30)
```

**Step 4: TLD (Top-Level Domain) DNS Server**

`.com`, `.org`, `.ai`, `.app` 등 각 TLD마다 전담 서버가 있다.
```
질문: "example.com의 IP는?"
.com TLD 서버: "나는 모르지만, example.com을 관리하는 네임서버를 알려줄게"
              → ns1.example.com (93.184.216.34)
```

**Step 5: Authoritative DNS Server**

해당 도메인의 실제 DNS 레코드를 가지고 있는 서버.
```
질문: "example.com의 IP는?"
Authoritative 서버: "93.184.216.34야" (A 레코드)
```

### 전체 흐름 다이어그램

```
브라우저: "example.com의 IP?"
    │
    ↓
[1] 로컬 캐시 (브라우저 → OS → hosts → 공유기)
    │ miss
    ↓
[2] Recursive Resolver (8.8.8.8 등)
    │ 캐시 miss
    ├──→ [3] Root Server (a.root-servers.net)
    │         "→ .com TLD 서버는 a.gtld-servers.net"
    │
    ├──→ [4] TLD Server (.com) (a.gtld-servers.net)
    │         "→ example.com 네임서버는 ns1.example.com"
    │
    └──→ [5] Authoritative Server (ns1.example.com)
              "→ example.com = 93.184.216.34"

    ↓
Recursive Resolver가 결과를 캐시 (TTL 동안)
    ↓
브라우저에 IP 반환
```

### DNS 레코드 타입

```
A       → IPv4 주소          example.com → 93.184.216.34
AAAA    → IPv6 주소          example.com → 2606:2800:220:1:...
CNAME   → 다른 도메인 별칭    www.example.com → example.com
MX      → 메일 서버          example.com → mail.example.com
NS      → 네임서버           example.com → ns1.example.com
TXT     → 텍스트 (인증 등)    example.com → "v=spf1 include:..."
SOA     → 도메인 권한 정보    시리얼, 갱신 주기 등
SRV     → 서비스 위치         _sip._tcp.example.com → ...
```

### TTL (Time To Live)

```
example.com.  300  IN  A  93.184.216.34
              ↑
              TTL: 300초 (5분) 동안 캐시 유효

TTL이 짧으면: DNS 변경이 빠르게 반영, 하지만 DNS 조회 빈번
TTL이 길면: 캐시 효율 좋음, 하지만 변경 반영 느림
```

### TLD (Top-Level Domain) 종류

```
gTLD (Generic TLD) — 일반 최상위 도메인:
  전통: .com, .org, .net, .edu, .gov, .mil
  신규: .app, .dev, .io, .xyz, .blog, .shop ...

ccTLD (Country Code TLD) — 국가 코드 도메인:
  .kr (한국), .jp (일본), .uk (영국), .ai (앵귈라), .io (영국령 인도양)
  → .ai는 원래 앵귈라 국가 도메인이지만 AI 업계에서 인기

sTLD (Sponsored TLD) — 특정 커뮤니티 후원:
  .museum, .aero, .coop

Infrastructure TLD:
  .arpa (역방향 DNS 등 인프라용)
```

### 새로운 TLD는 어떻게 추가되는가?

ICANN (Internet Corporation for Assigned Names and Numbers)이 관리한다.

```
1. ICANN이 "New gTLD Program" 신청 기간을 연다 (2012년에 대규모 개방)

2. 신청 조건:
   - 신청비: $185,000 USD
   - 기술 인프라 증명 (DNS 서버 운영 능력)
   - 사업 계획서
   - 분쟁 해결 절차

3. 심사 과정:
   - 기술 심사 (DNS 운영 능력)
   - 재정 심사 (지속 운영 가능성)
   - 문자열 심사 (기존 TLD와 혼동 여부)
   - 이의 제기 기간 (다른 신청자나 기존 권리자)

4. 승인되면:
   - IANA (ICANN 산하)가 루트 존 파일에 새 TLD 추가
   - 전 세계 루트 서버에 전파
   - Registry 운영 시작 (도메인 등록 가능)

5. 운영 구조:
   Registry (TLD 운영자) → Registrar (판매 대행) → 사용자
   예: .app의 Registry는 Google, Registrar는 GoDaddy/Namecheap 등
```

실제 사례:
- `.app` → Google이 $25M에 낙찰 (2015), HTTPS 강제
- `.dev` → Google이 운영, HTTPS 강제
- `.io` → 영국령 인도양 제도 ccTLD, 테크 업계에서 인기
- `.ai` → 앵귈라 ccTLD, AI 붐으로 가격 급등
- `.xyz` → 저렴한 gTLD, 스타트업에서 인기 (abc.xyz = Alphabet)

### 루트 존 파일 — DNS의 최상위

```
루트 존 파일 (root zone file):
  모든 TLD와 그 네임서버 목록을 담고 있는 파일
  IANA가 관리, 전 세계 13개 루트 서버 그룹에 배포

  com.    172800  IN  NS  a.gtld-servers.net.
  org.    172800  IN  NS  a0.org.afilias-nst.info.
  ai.     172800  IN  NS  pch.whois.ai.
  app.    172800  IN  NS  ns-tld1.charlestonroadregistry.com.
  kr.     172800  IN  NS  b.dns.kr.
  ...

  새 TLD가 여기에 추가되면 → 전 세계에서 해당 도메인 사용 가능
```

## 8. HTTP의 탄생과 발전 (1989~2022)

TCP/IP로 컴퓨터 간 통신은 가능해졌지만, 문서를 공유하는 표준 방법이 없었다.

**Tim Berners-Lee (CERN, 1989):**
- 물리학자들이 논문과 데이터를 쉽게 공유할 수 있는 시스템을 제안
- 3가지를 발명: HTML (문서 형식), URL (문서 주소), HTTP (문서 전송 프로토콜)

### HTTP/0.9 (1991)

```
# 요청: 메서드와 경로만
GET /index.html

# 응답: HTML만 (헤더 없음)
<html>Hello World</html>
(연결 즉시 종료)
```
- GET 메서드만 존재, 헤더 없음, 상태 코드 없음
- HTML만 전송 가능, 요청마다 TCP 연결 생성/종료

### HTTP/1.0 (1996)

```
GET /index.html HTTP/1.0
Host: www.example.com
User-Agent: Mozilla/1.0

HTTP/1.0 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...</html>
(연결 종료)
```
- 헤더 추가 (Content-Type으로 이미지, 텍스트 등 구분 가능)
- 상태 코드 추가 (200, 404, 500...)
- POST, HEAD 메서드 추가
- 여전히 요청마다 TCP 연결 생성/종료

### HTTP/1.1 (1997)

```
GET /index.html HTTP/1.1
Host: www.example.com
Connection: keep-alive

HTTP/1.1 200 OK
Content-Type: text/html
Transfer-Encoding: chunked
Cache-Control: max-age=3600
```
- Keep-Alive 기본 (하나의 TCP 연결로 여러 요청)
- Chunked Transfer Encoding
- Cache-Control, ETag 등 캐싱 헤더
- PUT, DELETE, OPTIONS, PATCH 등 메서드 추가
- Host 헤더 필수 (하나의 IP에 여러 도메인)
- 문제: Head-of-Line Blocking (앞 요청이 느리면 뒤 요청도 대기)

### HTTP/2 (2015)

- 바이너리 프로토콜 (텍스트 → 바이너리 프레임)
- Multiplexing (하나의 TCP 연결에서 여러 스트림 동시 처리)
- Header Compression (HPACK)
- Server Push
- TCP 레벨 HoL Blocking은 여전히 존재

### HTTP/3 (2022)

- TCP 대신 QUIC (UDP 기반) 사용
- TCP 레벨 HoL Blocking 해결
- 0-RTT 연결 설정 가능
- 연결 마이그레이션 (Wi-Fi → LTE 전환 시 연결 유지)

## 전체 타임라인

```
1969  ARPANET 탄생
1974  TCP/IP 개념 제안 (Vint Cerf, Bob Kahn)
1981  IPv4 표준화 (RFC 791)
1983  DNS 도입, ARPANET이 TCP/IP로 전환
1989  Tim Berners-Lee가 WWW 제안
1991  HTTP/0.9
1995  SSL (HTTPS의 시작)
1996  HTTP/1.0
1997  HTTP/1.1
1998  IPv6 표준화 (RFC 2460)
2015  HTTP/2
2022  HTTP/3 (QUIC)
```
