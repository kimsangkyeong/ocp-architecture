# 심화 03 - NAS NIC egress(hostNetwork:false) 가능성 & Pod CIDR 실IP대역 구성

## 변경이력 (Change History)

| 버전 | 일자 | 작성자 | 작성/검증 | 변경 내용 |
|------|------------|-------------|-----------|-----------|
| v1.0 | 2026-09-11 | k.s.k & kiro | 1차 생성 + 자체 교차검증 + 링크 검증 | (Q1)CIS 모드별 NAS NIC egress+hostNetwork:false 가능성, (Q2)Pod CIDR 실IP대역(no-overlay/BGP) 구성 최초 작성 |

> 개념 근거는 [`../references/01-ovn-gateway-concepts.md`](../references/01-ovn-gateway-concepts.md), 다중 NIC/hostNetwork 배경은 [`01-brint-and-multinic-egress.md`](01-brint-and-multinic-egress.md), ipForwarding 심화는 [`02-ipforwarding-global-egress.md`](02-ipforwarding-global-egress.md), F5 연동은 [`../f5-cis-integration/01-f5-cis-clusterip-ovn-architecture.md`](../f5-cis-integration/01-f5-cis-clusterip-ovn-architecture.md), 출처는 [`../references/03-glossary-and-sources.md`](../references/03-glossary-and-sources.md) 참조.

---

## 0. 질문 요약

- **케이스 1:** F5-CIS(**ClusterIP mode**) + OVN(`routingViaHost:false`, `ipForwarding:Global`) 에서,
  하나의 Pod가 (a) 클러스터 내 다른 service/pod 호출 + (b) nodeport listen 결과를 (c) **NAS NIC를 경유하는 S3 IP** 로 통신·저장.
  이걸 **`spec.hostNetwork:false`** 로 가능한가? 그리고 **CIS(NodePort mode) + OVN(`routingViaHost:true`, `ipForwarding:Global`)** 이면 문제없을까?
- **케이스 2:** F5-CIS(ClusterIP mode) + OCP(**Pod CIDR를 machine IP처럼 real IP 대역으로 설정**)가 가능한가? 가능하면 OVN policy는?

---

## 1. 케이스 1 — NAS NIC egress를 hostNetwork:false로?

### 1.1 핵심 결론

| 조합 | (a) 다른 service/pod 호출 | (c) NAS NIC 경유 S3 IP 통신 | hostNetwork:false로 동시 성공? |
|------|:-------------------------:|:----------------------------:|:------------------------------:|
| **ClusterIP mode + `routingViaHost:false` + `ipForwarding:Global`** | O | **△ (거의 X에 가까움)** | **아니오 (신뢰성 있게는 불가)** |
| **NodePort mode + `routingViaHost:true` + `ipForwarding:Global`** | O | **O (성립 가능)** | **예 (조건부 성립)** — 단 "OVN이 자동 NIC 선택"이 아니라 "호스트 라우팅이 NAS NIC로 보냄" |

즉, 두 번째(사용자 추정)가 **방향은 맞습니다.** 다만 성립하는 이유는 "CIS를 NodePort로 바꿔서"가 아니라 **`routingViaHost:true`(Local GW) 로 Pod egress가 호스트 커널 라우팅을 타게 되어, 호스트의 정책 라우팅/멀티 NIC가 NAS NIC를 선택할 수 있기 때문**입니다.

### 1.2 왜 `routingViaHost:false`(Shared) 에서는 NAS NIC egress가 안 되나

Shared GW에서 Pod egress는 **OVS(br-ex)가 노드 IP 인터페이스로 직접** 내보냅니다. 호스트 커널 라우팅 테이블을 **경유하지 않으므로**, 호스트에 NAS NIC용 정책 라우팅/추가 경로를 넣어도 Pod egress가 그 경로를 "보지 못합니다." br-ex = 노드 기본 경로 NIC 하나로 나가는 구조라, S3가 NAS NIC 세그먼트에 있으면 도달 경로가 성립하지 않습니다.

- `ipForwarding:Global`은 "노드가 포워딩을 해준다"는 것이지 "egress가 호스트 라우팅 테이블을 경유하게 만든다"가 아닙니다. Shared GW에서는 애초에 트래픽이 호스트 라우팅을 안 타므로, **Global만으로 NAS NIC 선택이 되지 않습니다.**
- 근거: Shared GW는 egress가 호스트를 경유하지 않고 OVS가 노드 IP 인터페이스로 직접 출력. 원문 내용을 재구성함. [OKD/Red Hat: Configuring a gateway](https://docs.okd.io/4.19/networking/ovn_kubernetes_network_provider/configuring-gateway.html)

> Content was rephrased for compliance with licensing restrictions.

### 1.3 왜 `routingViaHost:true`(Local) 에서는 성립 가능한가

Local GW에서는 **모든 (ingress/egress) 트래픽이 OVN에 들어가기 전/나간 후 호스트 커널 네트워킹 스택을 경유**하며, **호스트 라우팅 테이블과 iptables가 평가**됩니다. 따라서 호스트에 "S3 대역 → NAS NIC" 정책 라우팅/경로를 넣어두면 **Pod egress가 그 경로를 따라 NAS NIC로 나갈 수 있습니다.** 이때 `ipForwarding:Global`은 노드가 그 인터페이스로 (비 OVN) 트래픽을 실제 포워딩하도록 허용하는 역할을 합니다.

- 근거: Local 모드에서는 클러스터 진입/이탈 전 모든 트래픽이 호스트 커널 스택으로 라우팅되어, ingress/egress 패킷에 대해 호스트 라우팅 테이블과 iptables가 평가됨. 원문 내용을 재구성함. [Red Hat Bugzilla 2042516 (Full Text)](https://bugzilla.redhat.com/show_bug.cgi?format=multiple&id=2042516)
- 근거: `routingViaHost:true`는 egress가 Pod를 호스팅하는 노드의 로컬 gateway를 통해 호스트를 경유하며 호스트 라우팅 테이블이 적용됨. 원문 내용을 재구성함. [Red Hat/OKD: Configuring a gateway](https://docs.okd.io/4.19/networking/ovn_kubernetes_network_provider/configuring-gateway.html)
- 근거: `ipForwarding` 기본 Restricted에서는 k8s 트래픽만 포워딩·그 외는 노드가 라우팅 안 함. 호스트가 관리 인터페이스로 포워딩하게 하려면 Global. 원문 내용을 재구성함. [Red Hat Solution 7053694](https://access.redhat.com/solutions/7053694)

> Content was rephrased for compliance with licensing restrictions.

### 1.4 중요한 오해 정정 — "OVN이 알아서 NIC 선택"은 아님

사용자가 기대하신 **"OVN network이 알아서 필요한 NIC를 선택하고 SNAT"** 는 기본(primary) 네트워크에서 **제공되지 않습니다**(이미 [`01-brint-and-multinic-egress.md`](01-brint-and-multinic-egress.md)에서 정리). Local GW에서 NAS NIC로 나가는 것은 **OVN이 판단한 것이 아니라 "호스트 커널의 라우팅 테이블"이 목적지(S3 IP)를 보고 NAS NIC로 보낸 것**입니다. 따라서 다음 전제가 반드시 필요합니다.

- 노드(호스트)에 **NAS NIC가 실제로 존재**하고, **"S3 IP/대역 → NAS NIC(via NAS gateway)" 경로**가 호스트 라우팅/정책 라우팅에 설정되어 있을 것.
- 응답 대칭성(NAS 쪽에서 돌아오는 경로)과 `rp_filter` 등 확인.
- `ipForwarding:Global` 로 해당 인터페이스 포워딩 허용.

### 1.5 CIS 모드(ClusterIP vs NodePort)는 이 문제의 "원인"이 아니다

- 케이스 1에서 (a)(b)(c)는 **모두 "Pod가 시작하는 아웃바운드"** 이거나 **"노드가 받는 인바운드(nodeport listen)"** 입니다. **CIS 모드(ClusterIP/NodePort)는 F5→백엔드 로드밸런싱 방식(풀 멤버가 Pod IP냐 노드 IP냐)** 을 정할 뿐, **Pod의 아웃바운드 egress 경로(→S3)** 를 직접 바꾸지 않습니다.
- 따라서 "NodePort로 바꿔서 됐다"기보다는 **함께 바꾼 `routingViaHost:true`가 결정적 요인**입니다. NodePort mode 자체는 이 egress 문제와 직접 인과가 약합니다(다만 NodePort mode는 F5가 노드 IP로 붙으므로 F5 연동 관점의 return path 설계는 단순해질 수 있음).

> 근거: CIS pool-member-type(cluster/nodeport/auto)은 BIG-IP가 트래픽을 Pod IP로 보낼지 노드 IP로 보낼지를 정함. 원문 내용을 재구성함. [F5 clouddocs: BIG IP Networking with CIS](https://clouddocs.f5.com/containers/latest/userguide/config-options.html)

> Content was rephrased for compliance with licensing restrictions.

### 1.6 "동일 Pod에서 내부 호출 + 외부(S3) 호출 동시" 딜레마의 정공법

`routingViaHost:true`(Local) + `ipForwarding:Global` + 호스트 정책 라우팅(S3→NAS NIC) 조합이면, **hostNetwork:false 상태로도** (a)내부 service/pod 호출(오버레이)과 (c)NAS NIC 경유 S3 호출을 **동시에** 만족할 여지가 생깁니다. 이는 [`02-ipforwarding-global-egress.md`](02-ipforwarding-global-egress.md) §5에서 언급한 "hostNetwork on/off 어느 쪽도 동시 만족이 어렵던 문제"의 근본 해법에 해당합니다(EgressIP 미사용 전제와도 부합).

단, **반드시 §1.7 절차로 실제 경로를 검증**해야 확정입니다(호스트 라우팅/대칭성/방화벽 변수 때문).

### 1.7 확정 진단 절차 (hostNetwork:false, Local GW + Global 적용 후)

```bash
# 1) Pod가 S3 IP로 SYN을 보내는지 (Pod netns)
#    src=PodIP → dst=<S3_IP>:<port>
# 2) 노드의 NAS NIC 에서 SYN이 나가는지
tcpdump -ni <nas_nic> host <S3_IP>
#    기대: src=노드/NAS NIC IP(SNAT 후) → dst=<S3_IP>
# 3) 노드 호스트 라우팅 확인 (S3 대역이 NAS NIC via NAS gateway로 가는지)
oc debug node/<node> -- chroot /host ip route get <S3_IP>
# 4) 동시에 내부 pod 호출(오버레이)이 정상인지 (Pod→ClusterIP/PodIP)
```

판정: (2)에서 NAS NIC로 SYN이 나가고 (4) 내부 통신도 정상이면 목표 달성. (2)에서 안 나가면 호스트 라우팅/포워딩(Global)·정책 라우팅 재점검.

---

## 2. 케이스 2 — Pod CIDR을 machine IP처럼 "real IP 대역"으로 구성

### 2.1 결론: 가능합니다 (단, 방식과 제약이 있음)

Pod IP를 SNAT 없이 **외부(provider) 네트워크에서 직접 라우팅 가능한 "실 IP"처럼** 다루는 구성은 지원됩니다. 대표적으로 다음 2가지입니다.

| 방식 | 요지 | Pod IP 외부 직접 도달 | 성숙도 |
|------|------|:---------------------:|--------|
| **A. Route Advertisements (BGP, PodNetwork 광고)** | OVN-K/FRR가 BGP로 **Pod 서브넷을 provider 네트워크에 광고**. 노드가 next-hop. `routingViaHost:false`로도 수동 경로설정 없이 도달 | O | GA 방향(4.20~4.22 문서화) |
| **B. No-Overlay mode (underlay routing, BGP)** | 기본망/CUDN의 **Geneve 캡슐화를 끄고** underlay로 직접 라우팅. `outboundSNAT:Disabled` 시 Pod IP가 그대로 노출 | O | **Tech Preview(4.22)** |

- 근거(A): Route advertisements로 기본 Pod 네트워크/사용자 정의 네트워크 경로(EgressIP 포함)를 provider 네트워크와 상호 광고 가능. provider 네트워크에서 광고된 IP로 직접 도달(및 그 반대) 가능. 예전엔 `routingViaHost:true`+노드별 수동 경로로 근사했으나, route advertisements로 `routingViaHost:false` 에서 매끄럽게 달성. 원문 내용을 재구성함. [Red Hat OCP 4.22: Route advertisements](https://docs.redhat.com/en/documentation/OpenShift_container_platform/4.22/html/advanced_networking/route-advertisements)
- 근거(A): BGP 광고로 cloud provider 라우터가 "노드가 pod/service 네트워크의 next-hop"임을 알게 됨. 원문 내용을 재구성함. [Red Hat OCP 4.22: BGP routing](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/advanced_networking/bgp-routing)
- 근거(B): No-overlay 모드는 기본망의 캡슐화를 끄고 BGP 학습 경로로 Pod 트래픽을 노드 간 전달. `outboundSNAT`는 Pod IP가 외부망에서 라우팅 불가하면 Enabled, underlay가 Pod IP를 직접 라우팅 가능하면 Disabled. LGW/SGW 모두 지원. 원문 내용을 재구성함. [OKD 4.22: no-overlay mode BGP routing](https://docs.okd.io/4.22/networking/advanced_networking/bgp_routing/no-overlay-mode-bgp-routing.html)

> Content was rephrased for compliance with licensing restrictions.

### 2.2 이때 OVN policy 설정

#### 방식 A: Route Advertisements (권장, `routingViaHost:false` 유지)

```yaml
# 1) CNO: FRR provider 활성 + route advertisements 활성
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  additionalRoutingCapabilities:
    providers:
      - FRR
  defaultNetwork:
    type: OVNKubernetes
    ovnKubernetesConfig:
      routeAdvertisements: Enabled
      gatewayConfig:
        routingViaHost: false     # 광고 사용 시 false로도 수동경로 불필요
        ipForwarding: Restricted  # (BGP 광고로 도달하므로 보통 Restricted 유지 가능)
```

```yaml
# 2) FRRConfiguration (외부 BGP peer)
apiVersion: frrk8s.metallb.io/v1beta1
kind: FRRConfiguration
metadata:
  name: external-bgp
  namespace: openshift-frr-k8s
  labels: { network: default }
spec:
  bgp:
    routers:
      - asn: 64512
        neighbors:
          - address: <외부라우터_IP>
            asn: 64512
```

```yaml
# 3) RouteAdvertisements (Pod 서브넷을 광고)
apiVersion: k8s.ovn.org/v1
kind: RouteAdvertisements
metadata: { name: default }
spec:
  advertisements: [ PodNetwork ]
  networkSelectors:
    - networkSelectionType: DefaultNetwork
  frrConfigurationSelector:
    matchLabels: { network: default }
  nodeSelector: {}
```

> 핵심: `routingViaHost:false` 를 유지하면서 BGP로 Pod 서브넷을 광고 → 외부에서 Pod IP로 직접 도달. 과거처럼 `routingViaHost:true`+노드별 수동 static route를 넣을 필요가 없습니다.

#### 방식 B: No-Overlay (Tech Preview, Pod IP를 SNAT 없이 노출)

```yaml
apiVersion: operator.openshift.io/v1
kind: Network
metadata: { name: cluster }
spec:
  additionalRoutingCapabilities:
    providers: [ FRR ]
  defaultNetwork:
    type: OVNKubernetes
    ovnKubernetesConfig:
      routeAdvertisements: Enabled
      transport: NoOverlay
      noOverlayConfig:
        outboundSNAT: Disabled   # underlay가 Pod IP 직접 라우팅 시 Disabled
        routing: Unmanaged       # 외부 BGP peer 사용 (또는 Managed=노드 풀메시)
```

> No-overlay는 **Tech Preview**(프로덕션 SLA 미보장), 그리고 **EgressIP/EgressService/IPsec/멀티캐스트/다중 외부 게이트웨이 미지원**, layer3 전용 등 제약이 있습니다. 도입 전 반드시 지원 상태/제약을 확인하세요. 근거: [OKD 4.22: no-overlay mode BGP routing](https://docs.okd.io/4.22/networking/advanced_networking/bgp_routing/no-overlay-mode-bgp-routing.html)

### 2.3 F5-CIS(ClusterIP mode) 관점에서의 의미

- **방식 A/B로 Pod IP가 real routable가 되면**, F5 BIG-IP는 CIS static route(`podCIDR via nodeIP`)에 의존하지 않고 **BGP fabric으로 Pod IP에 직접 도달**할 수 있습니다(또는 BIG-IP도 BGP peer로 참여). ClusterIP mode(풀 멤버=Pod IP)와 자연스럽게 맞습니다.
- 단, **F5도 해당 라우팅 도메인/BGP에 참여**하거나 provider 라우터를 통해 Pod 서브넷 경로를 학습해야 합니다. 그렇지 않으면 여전히 CIS static route 방식([F5 연동 문서](../f5-cis-integration/01-f5-cis-clusterip-ovn-architecture.md))이 필요합니다.
- return path/SNAT: Pod IP가 real IP로 노출되면 SNAT 없이 대칭 라우팅이 성립할 수 있으나(방식 B `outboundSNAT:Disabled`), F5 VIP 쪽 SNAT(AutoMap) 여부는 별도 설계 사항입니다.

### 2.4 주의 — real IP 대역 설계 시

- Pod 서브넷/노드 서브넷이 **기존 네트워크와 중복되지 않아야** 하고 BGP fabric에 광고 가능해야 함(IP 계획). 근거: [OKD 4.22: no-overlay](https://docs.okd.io/4.22/networking/advanced_networking/bgp_routing/no-overlay-mode-bgp-routing.html)
- 멀티 NIC 노드에서 "어느 노드 IP가 next-hop인가"는 여전히 중요(케이스 1과 연결). BGP/광고 설계에서 next-hop 인터페이스를 명확히.

---

## 3. 종합 요약 (Q1 · Q2)

| 질문 | 결론 |
|------|------|
| Q1) ClusterIP + `rvh:false` + `Global` 로 hostNetwork:false에서 NAS NIC egress? | **신뢰성 있게는 불가.** Shared GW는 egress가 호스트 라우팅을 안 타므로 Global이어도 NAS NIC 선택 불가 |
| Q1) NodePort + `rvh:true` + `Global` 이면? | **성립 가능(방향 맞음).** 단 결정 요인은 NodePort가 아니라 **`rvh:true`(Local GW)로 egress가 호스트 라우팅을 경유**하는 것. 호스트에 S3→NAS NIC 정책 라우팅 + Global 필요. §1.7로 검증 |
| Q1) "OVN이 자동 NIC 선택"인가? | **아님.** 호스트 커널 라우팅이 선택. 노드에 NAS NIC/경로가 실재해야 함 |
| Q2) Pod CIDR을 real IP 대역으로 구성 가능? | **가능.** (A)Route Advertisements(BGP, `rvh:false` 유지) 또는 (B)No-Overlay(Tech Preview, `outboundSNAT:Disabled`) |
| Q2) OVN policy? | A: `additionalRoutingCapabilities:FRR` + `routeAdvertisements:Enabled` + FRRConfiguration + RouteAdvertisements(PodNetwork), `rvh:false`. B: `transport:NoOverlay` + `noOverlayConfig(outboundSNAT:Disabled, routing:Unmanaged)` |

> 권고: Q1은 **CIS 모드 변경보다 `routingViaHost:true`(Local GW) + `ipForwarding:Global` + 호스트 정책 라우팅**이 본질적 해법이며 hostNetwork:false로 동시(내부+NAS) 통신을 노려볼 수 있습니다(반드시 tcpdump 검증). Q2는 EgressIP를 원치 않는 전제에서 **Route Advertisements(BGP)** 가 `routingViaHost:false` 를 유지하며 Pod IP를 real routable로 만드는 가장 정공법입니다.

---

## 4. 출처 (링크 검증 완료)

| 출처 | 링크 | 상태 |
|------|------|------|
| Red Hat OCP 4.22: Route advertisements | https://docs.redhat.com/en/documentation/OpenShift_container_platform/4.22/html/advanced_networking/route-advertisements | 페이지 유효(스크립트 403 봇차단) |
| Red Hat OCP 4.22: BGP routing | https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/advanced_networking/bgp-routing | 페이지 유효(스크립트 403 봇차단) |
| OKD 4.22: no-overlay mode BGP routing | https://docs.okd.io/4.22/networking/advanced_networking/bgp_routing/no-overlay-mode-bgp-routing.html | 200 OK |
| OKD: About BGP routing | https://docs.okd.io/latest/networking/advanced_networking/bgp_routing/about-bgp-routing.html | 200 OK |
| OKD: MetalLB symmetric routing (멀티 인터페이스 소스 IP 이슈) | https://docs.okd.io/4.18/networking/ingress_load_balancing/metallb/metallb-configure-return-traffic.html | 200 OK |
| Red Hat/OKD: Configuring a gateway | https://docs.okd.io/4.19/networking/ovn_kubernetes_network_provider/configuring-gateway.html | 200 OK |
| Red Hat Bugzilla 2042516 (Local=호스트 커널 경유) | https://bugzilla.redhat.com/show_bug.cgi?format=multiple&id=2042516 | 200 OK |
| Red Hat Solution 7053694 (ipForwarding Global) | https://access.redhat.com/solutions/7053694 | 200 OK |
| F5 clouddocs: BIG IP Networking with CIS (모드) | https://clouddocs.f5.com/containers/latest/userguide/config-options.html | 200 OK |

> docs.redhat.com은 자동화 클라이언트에 403(봇 차단)이나 브라우저/조회도구로는 정상(끊어진 링크 아님). 동일 내용의 무봇차단 미러로 OKD 링크를 병행 제시했습니다.
