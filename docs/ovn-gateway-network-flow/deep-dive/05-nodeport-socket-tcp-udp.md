# 심화 05 - TCP/UDP 소켓과 NodePort Service / 앱이 노드포트를 직접 소유하는 사례

## 변경이력 (Change History)

| 버전 | 일자 | 작성자 | 작성/검증 | 변경 내용 |
|------|------------|-------------|-----------|-----------|
| v1.0 | 2026-09-11 | k.s.k & kiro | 1차 생성 + 자체 교차검증 + 링크 검증 | TCP/UDP 소켓의 NodePort Service 동작(수신/응답/소스IP), 내부 pod 호출 병행, 앱이 노드포트 직접 소유(hostNetwork/hostPort) 사례 최초 작성 |

> 배경(오해 교정)은 F5 문서 [`../f5-cis-integration/01-...clusterip....md`](../f5-cis-integration/01-f5-cis-clusterip-ovn-architecture.md) §5-B, [`../f5-cis-integration/02-...nodeportlocal....md`](../f5-cis-integration/02-f5-cis-nodeportlocal-antrea-architecture.md) §4-B 참조. 출처는 [`../references/03-glossary-and-sources.md`](../references/03-glossary-and-sources.md).

---

## 0. 질문 요약

1. **TCP/UDP 소켓 통신 프로그램**도 `kind: Service, type: NodePort` 로 노출하여 **send/receive** 를 할 수 있는가?
2. 그 Pod가 동시에 **cluster 내 다른 service/pod 와 메시지 호출**(클라이언트로서)도 할 수 있는가?
3. **앱이 노드 포트를 직접 소유**하는 프로그램의 사례.

**결론 미리:**
- (1) **가능.** NodePort는 TCP/UDP 모두 지원. 앱은 `targetPort`(예 8080)만 listen하면 되고, 노드포트(30000)는 kube-proxy가 열어 DNAT로 전달. **단 UDP는 소스IP/세션 특성상 주의점**이 있음(§2).
- (2) **가능.** NodePort 노출(=inbound 경로)과, Pod가 클라이언트로서 내부 ClusterIP/Pod를 호출(=outbound)하는 것은 **완전히 별개**로 동시에 성립.
- (3) `hostNetwork: true` 또는 `hostPort` 를 쓰면 앱이 노드 포트를 직접 소유(§4).

---

## 1. NodePort Service는 TCP/UDP 소켓에 어떻게 동작하나

### 1.1 기본 모델

- Service의 `protocol` 필드로 **TCP 또는 UDP**(또는 SCTP)를 지정합니다. NodePort는 그 프로토콜의 노드포트를 엽니다.
- 앱(Pod)은 **자기 포트(targetPort)만 listen/bind** 합니다. 외부에서 `nodeIP:nodePort` 로 오면 kube-proxy가 **DNAT** 하여 `PodIP:targetPort` 로 전달합니다.
- 즉 소켓 프로그램 관점에서는 **평범하게 `bind(0.0.0.0:8080)` + `recv/send`(TCP면 accept 후, UDP면 recvfrom/sendto)** 를 하면 됩니다. 노드포트 개방/전달은 쿠버네티스가 담당.

### 1.2 TCP/UDP Service 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: udp-echo
spec:
  type: NodePort
  selector: { app: udp-echo }
  ports:
    - name: udp
      protocol: UDP          # ★ UDP 지정
      port: 9000
      targetPort: 9000       # Pod 앱이 bind 하는 포트
      nodePort: 30900        # 노드에서 열리는 UDP 포트
    # TCP도 필요하면 별도 포트로 추가
    - name: tcp
      protocol: TCP
      port: 9000
      targetPort: 9000
      nodePort: 30901
```

- 앱: `bind(0.0.0.0:9000)` (UDP: `recvfrom`/`sendto`, TCP: `listen`/`accept`/`recv`/`send`). `hostNetwork` 불필요.
- 외부/F5: `nodeIP:30900`(UDP) 또는 `nodeIP:30901`(TCP) 로 접속.

---

## 2. 소스 IP / 응답(send-back) 특성 — 특히 UDP 주의

### 2.1 기본(`externalTrafficPolicy: Cluster`)은 SNAT

NodePort로 들어온 패킷은 **기본적으로 SNAT** 됩니다(소스 IP가 수신 노드 IP로 치환). kube-proxy 동작 단계:

1. 클라이언트가 `node2:nodePort` 로 패킷 전송
2. node2가 소스 IP를 **자신의 IP로 SNAT**
3. node2가 목적지 IP를 **Pod IP로 DNAT**
4. 패킷이 (필요 시 다른 노드의) 엔드포인트 Pod로 라우팅
5. Pod의 응답이 node2로 돌아오고
6. node2가 클라이언트로 응답 전송

- 근거: `type=NodePort` Service로 보낸 패킷은 기본적으로 SNAT됨. 클라이언트→node2:nodePort, node2가 소스 IP를 자신 IP로 SNAT, 목적지를 Pod IP로 DNAT, 응답은 node2를 거쳐 클라이언트로. 원문 내용을 재구성함. [Kubernetes: Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/)

> Content was rephrased for compliance with licensing restrictions.

의미(소켓 앱 관점):
- **앱이 보는 소스 IP = 클라이언트 실제 IP가 아니라 노드 IP**(SNAT 때문). UDP 서버가 `recvfrom` 으로 얻는 peer 주소가 노드 IP가 됩니다.
- **응답(send)은 반드시 recvfrom으로 받은 peer 주소로 sendto** 해야 합니다. conntrack이 역방향(역-DNAT/역-SNAT)을 처리해 클라이언트로 되돌아갑니다. **소켓을 "connected UDP"로 고정하거나 임의 소스로 응답하면** conntrack 매핑이 어긋나 응답이 유실될 수 있습니다(UDP 특유 이슈).

### 2.2 클라이언트 실제 IP가 필요하면 `externalTrafficPolicy: Local`

- `service.spec.externalTrafficPolicy: Local` 로 설정하면 kube-proxy가 **로컬 엔드포인트로만** 프록시하고 다른 노드로 포워딩하지 않아 **원본 클라이언트 소스 IP가 보존**됩니다. 대신 **로컬 엔드포인트가 없는 노드로 온 패킷은 드롭**됩니다(그 노드에는 Pod가 없으므로).
- 근거: `externalTrafficPolicy: Local` 설정 시 kube-proxy는 로컬 엔드포인트로만 프록시하고 타 노드로 포워딩하지 않아 원본 소스 IP를 보존. 로컬 엔드포인트가 없으면 패킷 드롭. 원문 내용을 재구성함. [Kubernetes: Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/)

> Content was rephrased for compliance with licensing restrictions.

| 정책 | 소스 IP | 크로스노드 | 용도 |
|------|---------|:---------:|------|
| `Cluster`(기본) | 노드 IP(SNAT) | O(어느 노드로 와도 분산) | 가용성 우선. 클라이언트 IP 불필요할 때 |
| `Local` | **클라이언트 실제 IP 보존** | X(로컬 Pod만) | 클라이언트 IP 기반 ACL/로깅/UDP 소스식별 필요할 때 |

> UDP 소켓에서 **peer(클라이언트) 실제 IP로 식별/응답**이 중요하면 `Local` + (F5/외부 LB가 Pod 있는 노드로 보내도록) 구성이 필요합니다.

---

## 3. 동시에 "내부 service/pod 클라이언트 호출"도 되는가 — 예

- NodePort로 **들어오는(inbound)** 것과, 그 Pod가 **나가서(outbound) 내부 ClusterIP/Pod 를 호출**하는 것은 **서로 다른 방향의 독립 경로**입니다. 하나가 다른 하나를 막지 않습니다.
- 내부 호출(Pod → ClusterIP/PodIP)은 클러스터 CNI(오버레이)로 처리되며, **ClusterIP 내부 접근은 소스 IP가 보존**(SNAT 없음)됩니다. 앞선 시나리오1/2 문서 참조.
- 근거: 클러스터 내부에서 ClusterIP로 접근 시 client_address는 항상 클라이언트 Pod IP(같은 노드든 다른 노드든). 원문 내용을 재구성함. [Kubernetes: Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/)

> Content was rephrased for compliance with licensing restrictions.

패킷 관점(동시 동작):
```
[inbound]  외부 → nodeIP:30900(UDP) --(kube-proxy DNAT)--> PodIP:9000   (앱 recvfrom)
[outbound] 앱 --(connect/sendto)--> 다른 ClusterIP:port / PodIP:port    (오버레이, src=PodIP)
```
- 두 소켓(수신용 bind 소켓 / 송신용 client 소켓)은 앱 내에서 별개로 열면 됩니다. 서버 소켓(9000)과 클라이언트 소켓(임의 소스 포트)이 공존합니다.

### 3.1 예: UDP 에코 + 내부 호출 (Python 개념 코드)

```python
import socket

# (1) NodePort로 들어오는 UDP 수신 서버 소켓
srv = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
srv.bind(("0.0.0.0", 9000))          # Pod netns eth0:9000 (노드 30900은 kube-proxy가 담당)

# (2) 내부 service 호출용 UDP 클라이언트 소켓 (별개)
cli = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

while True:
    data, peer = srv.recvfrom(65535)  # peer = (SNAT된 노드 IP, port)  [Cluster 정책 시]
    # 내부 다른 service(ClusterIP)로 메시지 호출
    cli.sendto(data, ("other-svc.myns.svc.cluster.local"  # DNS→ClusterIP
                      , 8080)) if False else None
    cli.sendto(data, ("10.128.10.5", 8080))   # 예: 내부 ClusterIP:8080
    # 원 요청자에게 응답 (반드시 recvfrom의 peer로)
    srv.sendto(b"ack:" + data, peer)
```

> 핵심: 수신 응답은 `recvfrom`이 준 `peer` 로만 `sendto`. 내부 호출은 별도 `cli` 소켓으로. 두 경로가 독립적으로 동작합니다.

---

## 4. 앱이 "노드 포트를 직접 소유"하는 사례

Service/kube-proxy를 거치지 않고 **앱 프로세스가 노드의 포트를 직접 bind** 해야 하는 경우입니다. 두 가지 방법이 있습니다.

### 4.1 방법 A — `hostNetwork: true` (노드 netns 직접 사용)

Pod가 노드 네트워크 네임스페이스를 그대로 쓰므로, `bind(0.0.0.0:30000)` 이 **노드의 모든 NIC(service NIC 포함)** 에 바인딩됩니다. 앱이 노드포트를 실질적으로 소유.

```yaml
apiVersion: v1
kind: Pod
metadata: { name: udp-collector }
spec:
  hostNetwork: true            # ★ 노드 netns 사용
  dnsPolicy: ClusterFirstWithHostNet   # hostNetwork 시 클러스터 DNS 쓰려면 필요
  containers:
    - name: collector
      image: my-udp-collector:latest
      ports:
        - containerPort: 30000
          hostPort: 30000       # 문서화/스케줄링 힌트 (hostNetwork면 사실상 동일)
          protocol: UDP
```

- 앱: `bind(0.0.0.0:30000)` → 노드의 30000/UDP를 직접 수신. 소스 IP도 **클라이언트 실제 IP 그대로**(kube-proxy DNAT/SNAT 미개입).
- 사례: **syslog(UDP 514) 수집기, SNMP trap(UDP 162) 수신기, NetFlow/sFlow collector, GTP/RADIUS 등 통신 프로토콜 게이트웨이, DHCP relay** 등 "특정 고정 포트로 대량 UDP를 노드에서 직접 받아야 하는" 워크로드.
- 주의: Pod=노드 netns → **격리 약화, 포트 충돌(노드당 1개), Pod IP=노드 IP**. 노드당 하나만 뜨도록 DaemonSet + 안티어피니티 권장. 내부 pod 호출(outbound)은 여전히 가능하나 소스가 노드 IP.

### 4.2 방법 B — `hostPort` (특정 컨테이너 포트만 노드에 매핑)

`hostNetwork` 없이(Pod는 자체 netns 유지) **특정 컨테이너 포트만 노드 포트에 매핑**합니다. CNI portmap 플러그인이 노드포트→PodIP:port DNAT를 설정합니다.

```yaml
apiVersion: v1
kind: Pod
metadata: { name: edge-app }
spec:
  containers:
    - name: app
      image: my-app:latest
      ports:
        - containerPort: 9000
          hostPort: 30000       # ★ 노드의 30000 → 이 Pod의 9000
          protocol: TCP
```

- 앱: `bind(0.0.0.0:9000)`(Pod netns). 노드 30000으로 오면 portmap이 PodIP:9000으로 DNAT.
- `hostNetwork`와 달리 **Pod는 자체 IP 유지**(격리 유지). 하지만 **노드당 그 hostPort는 하나의 Pod만** 사용 가능(스케줄 제약).
- 사례: **엣지/게이트웨이 파드, 노드 로컬로만 고정 포트가 필요한 에이전트**. 다만 일반 서비스 노출은 NodePort Service가 더 표준적.

### 4.3 방법 비교

| 방법 | Pod IP | 노드포트 소유 | 소스 IP(수신) | 격리 | 노드당 개수 | 대표 사례 |
|------|--------|:-------------:|---------------|:----:|:-----------:|-----------|
| **NodePort Service** | Pod 자체 | kube-proxy | 노드 IP(SNAT, 기본) / Local이면 실IP | 유지 | 제한 없음(포트는 클러스터 공용) | 일반 TCP/UDP 서비스 노출 |
| **hostNetwork:true** | = 노드 IP | **앱** | **클라이언트 실IP** | 약함 | 1(포트당) | syslog/SNMP/NetFlow collector |
| **hostPort** | Pod 자체 | 앱(portmap DNAT) | 클라이언트 실IP(대개) | 유지 | 1(hostPort당) | 엣지 게이트웨이/노드로컬 에이전트 |

---

## 5. 요약 / 권고

1. **TCP/UDP 소켓 + NodePort Service:** **가능.** 앱은 `targetPort`만 bind/listen, 노드포트는 kube-proxy가 개방·DNAT. `hostNetwork` 불필요.
2. **UDP 주의:** 기본(`Cluster`)은 SNAT라 앱이 보는 소스 IP가 노드 IP. 응답은 `recvfrom` peer로 `sendto`. 클라이언트 실IP 필요하면 `externalTrafficPolicy: Local`(로컬 엔드포인트만, 크로스노드 X).
3. **내부 호출 병행:** inbound(NodePort)와 outbound(내부 ClusterIP/Pod 호출)는 독립. 동시에 가능. 내부 ClusterIP 접근은 소스 IP 보존.
4. **앱이 노드포트 직접 소유:** `hostNetwork:true`(노드 전체 포트, 실IP 수신) 또는 `hostPort`(특정 포트만 매핑, Pod IP 유지). syslog/SNMP/NetFlow 등 UDP 수집기가 대표 사례.

---

## 6. 출처 (링크 검증 완료)

| 출처 | 링크 | 상태 |
|------|------|------|
| Kubernetes: Using Source IP (NodePort SNAT / externalTrafficPolicy) | https://kubernetes.io/docs/tutorials/services/source-ip/ | 200 OK |

> hostNetwork/hostPort 소켓 바인딩 원리는 표준 Kubernetes/Linux netns 동작이며, 위 소스 및 앞선 F5 문서(§5-B/§4-B)와 정합합니다.
