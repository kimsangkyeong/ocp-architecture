# 심화 04 - Multus 보조 네트워크 (기술 원리 · 구성 · 패킷 분석)

## 변경이력 (Change History)

| 버전 | 일자 | 작성자 | 작성/검증 | 변경 내용 |
|------|------------|-------------|-----------|-----------|
| v1.0 | 2026-09-11 | k.s.k & kiro | 1차 생성 + 자체 교차검증 + 링크 검증 | Multus 개념/기술원리, NAD 구성 예시, IP별 NIC 라우팅 원리 및 패킷 분석 최초 작성 |
| v1.1 | 2026-09-11 | k.s.k & kiro | Q&A 섹션 추가 + 링크 검증 | 3-A절 추가: 보조 NIC 고유 IP 필요 여부, 추가 IP 없이 노드 IP SNAT 대안(hostNetwork / CNI egress+정책라우팅) 비교 |

> 개념 근거는 [`../references/01-ovn-gateway-concepts.md`](../references/01-ovn-gateway-concepts.md), 다중 NIC egress 배경은 [`01-brint-and-multinic-egress.md`](01-brint-and-multinic-egress.md) / [`03-nasnic-egress-and-routable-podcidr.md`](03-nasnic-egress-and-routable-podcidr.md), 출처는 [`../references/03-glossary-and-sources.md`](../references/03-glossary-and-sources.md) 참조.

---

## 0. 왜 Multus가 필요한가 (직전 논의 연결)

앞선 문서들에서 정리한 핵심 제약:

- 기본(primary) 네트워크의 Pod는 **인터페이스가 하나(`eth0`)** 이고, 그 netns의 **기본 경로(default route)는 클러스터 CNI(OVN-K/Antrea)** 를 향합니다.
- 따라서 Pod가 목적지 IP만 보고 "특정 NIC(예: NAS NIC)로 알아서 나가게" 하는 것은 **기본 네트워크만으로는 불가능**합니다. (CNI가 목적지 기반으로 물리 NIC를 자동 선택하지 않음)

**Multus** 는 이 제약을 푸는 정공법입니다. Pod에 **두 번째(이후) 네트워크 인터페이스(`net1`, `net2`...)** 를 추가로 붙여, 특정 트래픽을 그 인터페이스(=특정 물리 NIC 경로)로 내보낼 수 있게 합니다. **`hostNetwork: false`** 를 유지하면서요.

---

## 1. Multus란 무엇인가 (기술 설명)

### 1.1 정의

**Multus CNI** 는 "메타 플러그인(meta-plugin) / CNI 멀티플렉서"입니다. Kubernetes Pod는 기본적으로 인터페이스가 하나지만, Multus는 **여러 CNI 플러그인을 조합**해 **하나의 Pod에 복수의 네트워크 인터페이스**를 붙입니다.

- 기본(primary/default) 네트워크: 기존 클러스터 CNI(OVN-Kubernetes/Antrea) — `eth0`. 클러스터 내부 통신·Service·기본 egress 담당.
- 보조(secondary/additional) 네트워크: Multus가 추가로 붙이는 `net1`, `net2`... — 특정 물리망/VLAN/전용 트래픽 담당.

- 근거: 설치 시 관리자는 Multus CNI로 대체 기본 보조 Pod 네트워크를 구성할 수 있고, ipvlan/macvlan/NAD 등 여러 CNI 플러그인을 조합해 Pod의 보조 네트워크로 사용 가능. 원문 내용을 재구성함. [Red Hat OCP 4.20: Multiple networks (single)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/multiple_networks/index)
- 근거: OpenShift에서 Multus 기반 보조 네트워크 구성은 CNO가 관리하며, Multus CNI는 NetworkAttachmentDefinition CR로 구현됨(Network Plumbing Working Group 주도). 원문 내용을 재구성함. [Red Hat Blog: Using the Multus CNI in OpenShift](https://www.redhat.com/en/blog/using-the-multus-cni-in-openshift)

> Content was rephrased for compliance with licensing restrictions.

### 1.2 핵심 오브젝트: NetworkAttachmentDefinition (NAD)

보조 인터페이스를 어떻게 붙일지는 **NetworkAttachmentDefinition(NAD)** CR로 정의합니다. NAD 안의 **CNI 설정(config)** 이 그 인터페이스가 어떻게 생성될지를 규정합니다.

- 근거: Pod에 보조 인터페이스를 붙이려면 붙이는 방법을 정의하는 구성을 만들어야 하며, 각 인터페이스는 NetworkAttachmentDefinition CR로 지정하고 CR 내부의 CNI 설정이 인터페이스 생성 방식을 정의함. 원문 내용을 재구성함. [OKD: Understanding multiple networks](https://docs.okd.io/latest/networking/multiple_networks/understanding-multiple-networks.html)

> Content was rephrased for compliance with licensing restrictions.

### 1.3 대표 보조 CNI 플러그인

| 플러그인 | 동작 | 대표 용도 |
|----------|------|-----------|
| **macvlan** | 물리 NIC에 **가상 MAC** 을 여러 개 부여, 각 Pod가 고유 MAC으로 물리망에 직접 참여 | 물리망(예: NAS/스토리지망) 직접 접속. ODF/스토리지 트래픽 분리 |
| **ipvlan** | 물리 NIC를 공유하되 **MAC은 공유, IP만 분리**(L2/L3 모드) | MAC 수 제한 환경, 스위치 포트 보안 |
| **host-device** | 노드의 **물리 NIC 자체(또는 VF)를 Pod로 이동** | 전용 NIC 독점 |
| **SR-IOV** | NIC의 VF를 Pod에 직접 할당(고성능/저지연) | 고성능 데이터플레인 |
| **bridge** | 노드의 리눅스 브리지에 연결 | 단순 L2 |
| **static / host-local / dhcp (IPAM)** | 보조 인터페이스의 IP 할당 방식 | - |

- 근거: macvlan 기반 보조 네트워크는 Pod가 물리 네트워크 인터페이스를 통해 다른 호스트/Pod와 통신하게 하며, 각 Pod에 고유 MAC 주소가 부여됨. 원문 내용을 재구성함. [Red Hat ODF: Creating Multus networks](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.18/html/managing_and_allocating_storage_resources/creating-multus-networks_rhodf)

> Content was rephrased for compliance with licensing restrictions.

---

## 2. 구성 방법 예시 (NAS NIC 경유 시나리오)

목표: Pod가 `eth0`(기본 CNI)로 클러스터 내부/일반 통신을 하면서, **S3(NAS 세그먼트, 예 `192.168.90.0/24`) 트래픽은 `net1`(NAS 물리 NIC `ens224` 경유)으로** 내보낸다. (노드에는 이미 NAS NIC와 라우팅이 있음)

### 2.1 NetworkAttachmentDefinition (macvlan + static IPAM + 라우트)

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: nas-macvlan
  namespace: my-app        # NAD는 사용하는 Pod와 동일 네임스페이스 권장
spec:
  config: '{
    "cniVersion": "0.3.1",
    "type": "macvlan",
    "master": "ens224",           
    "mode": "bridge",
    "ipam": {
      "type": "static",
      "addresses": [
        { "address": "192.168.90.51/24" }
      ],
      "routes": [
        { "dst": "192.168.90.0/24" }        
      ]
    }
  }'
```

- `master: ens224` = NAS 전용 물리 NIC. 이 NIC 경유로 S3망에 직접 참여.
- **중요:** `routes`에 **S3 대역만** 넣습니다(`dst: 192.168.90.0/24`). `0.0.0.0/0`(기본 경로)를 넣지 않는 것이 핵심입니다(§4.2에서 이유 설명).

### 2.2 Pod에 보조 네트워크 첨부

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-nas
  namespace: my-app
  annotations:
    k8s.v1.cni.cncf.io/networks: nas-macvlan     # 이 NAD를 net1로 붙임
spec:
  hostNetwork: false          # ★ hostNetwork 없이 달성
  containers:
    - name: app
      image: myapp:latest
      ports:
        - containerPort: 8080
```

첨부 후 Pod netns에는 인터페이스가 2개가 됩니다:
- `eth0` : 기본 CNI(예 `10.96.1.102`), default route → 클러스터 CNI
- `net1` : macvlan(`192.168.90.51`), 라우트 → `192.168.90.0/24`(NAS)

### 2.3 (선택) 기본 경로 자체를 보조망으로 바꾸는 경우 — 주의

macvlan 예시에서 `ipam.routes`에 `0.0.0.0/0`를 넣고 Pod annotation에 `"gateway": ["<gw>"]`를 지정하면 **기본 경로를 보조망으로 override** 할 수 있습니다. 하지만 이렇게 하면 **Pod 네트워크(클러스터 내부) 연결이 끊길 수 있습니다.**

- 근거: annotation의 `gateway`가 없으면 macvlan-conf에 기본 게이트웨이가 있어도 default route는 클러스터 네트워크 인터페이스(eth0)로 남음. `gateway`를 지정하면 macvlan의 것으로 override됨. 원문 내용을 재구성함. [openshift/multus-cni: macvlan-pod.yml 예시](https://raw.githubusercontent.com/openshift/multus-cni/main/examples/macvlan-pod.yml)
- 근거(트레이드오프): 보조망 default-route 지정 시 Pod 네트워크 연결이 사라질 수 있음(문서가 default route 변경의 영향을 언급). 원문 내용을 재구성함. [k8snetworkplumbingwg/multus-cni Issue #847](https://github.com/k8snetworkplumbingwg/multus-cni/issues/847)

> Content was rephrased for compliance with licensing restrictions.

> 권장: **특정 대역(S3)만 `net1`으로** 보내는 방식(§2.1)을 사용하고, 기본 경로(`eth0`→클러스터)는 그대로 두는 것이 안전합니다.

---

## 3. 어떻게 호출이 가능해지는가 — 원리

### 3.1 목적지 기반 라우팅(Policy/Destination routing)이 Pod netns 안에서 일어남

Multus가 `net1`을 붙이면서 **Pod netns의 라우팅 테이블**에 "S3 대역 → `net1`" 경로를 추가합니다. 그 결과:

- Pod가 **S3 IP(`192.168.90.x`)** 로 소켓을 열면 → netns 라우팅이 매칭 → **`net1`(macvlan, `ens224` 경유)** 으로 송신.
- Pod가 **클러스터 내부(`10.96.x`, ClusterIP) / 일반 외부** 로 열면 → default route(`eth0`) → 기존 CNI 경로.

즉, "OVN/Antrea가 목적지를 보고 NIC를 자동 선택"하는 게 아니라, **Pod 자신의 netns 라우팅 테이블이 목적지에 따라 인터페이스를 선택**하게 됩니다. 이것이 hostNetwork:false로도 IP별 NIC 라우팅이 되는 원리입니다.

### 3.2 macvlan의 소스 IP

- macvlan(`net1`)로 나가는 S3 트래픽의 **소스 IP는 `net1`의 IP(`192.168.90.51`)** 이며, 고유 MAC으로 물리망(NAS)에 직접 참여합니다. 노드 IP로의 SNAT가 아니라 **Pod의 macvlan IP가 그대로 소스**가 됩니다.
- 반대로 `eth0`로 나가는 일반 egress는 기존 CNI 규칙(노드 IP SNAT 등)을 따릅니다.

---

## 3-A. Q&A — 보조 NIC는 고유 IP가 꼭 필요한가? Node IP SNAT로 추가 IP 없이 안 되나?

### 3-A.1 결론 먼저

| 질문 | 답 |
|------|-----|
| Multus 보조 NIC(macvlan/ipvlan/host-device)는 **고유 IP가 필요한가?** | **예. 필요합니다.** 보조 인터페이스는 그 자체가 L2/L3 통신 종단이므로 **자신의 IP(IPAM으로 할당)** 가 있어야 정상적으로 IP 통신을 합니다. macvlan은 추가로 **고유 MAC** 까지 받습니다. |
| **Node IP로 SNAT** 해서 **추가 IP 없이** 통신 가능한가? | **Multus 방식으로는 사실상 불가**합니다. "노드 IP를 재사용해 SNAT로 나가기"는 **Multus가 아니라 호스트 egress 경로(hostNetwork 또는 CNI의 자체 egress)** 의 모델입니다. Multus 보조망은 "노드 IP 공유 SNAT"를 목적으로 설계된 것이 아닙니다. |

### 3-A.2 왜 보조 NIC는 고유 IP가 필요한가

- macvlan/ipvlan/host-device 등 Multus 보조 플러그인은 Pod에 **독립된 네트워크 인터페이스**를 만들고, 그 인터페이스로 L2/L3 통신을 합니다. 이 인터페이스가 패킷을 주고받으려면 **IPAM(static/DHCP/whereabouts)** 으로 **자신의 IP** 를 받아야 합니다.
- 특히 **macvlan** 은 물리 NIC 위에 **고유 MAC** 을 부여해 각 Pod가 물리망에 독립 호스트처럼 참여합니다. 즉 **고유 MAC + 고유 IP** 가 기본 전제입니다.
- 근거: macvlan 기반 보조 네트워크에 붙은 각 Pod에는 **고유 MAC 주소**가 부여되며, 물리 NIC를 통해 다른 호스트/Pod와 통신함. 원문 내용을 재구성함. [Red Hat: MicroShift Multiple networks (MACVLAN)](https://docs.redhat.com/en/documentation/red_hat_build_of_microshift/4.22/html/networking/multiple-networks)
- 근거: Multus 네트워크는 NAD로 구성하며, IPAM 플러그인(예 whereabouts)이 보조 인터페이스에 IP를 할당함. 원문 내용을 재구성함. [Red Hat ODF: Multus network configuration](https://docs.redhat.com/ja/documentation/red_hat_openshift_data_foundation/4.20/html/red_hat_openshift_data_foundation_architecture/multus-network-configuration_mcg)

> Content was rephrased for compliance with licensing restrictions.

> 참고: `ipam: none`(또는 정적으로 IP 미지정)으로 인터페이스를 만들 수는 있지만, 그 경우 **IP가 없어 일반적인 IP 통신(예 S3 TCP 세션)은 되지 않습니다.** (수동으로 IP를 넣거나, DPDK/L2 전용 등 특수 용도에 한정). 따라서 "S3로 저장" 같은 실제 L3 통신을 원하면 **보조 NIC에 IP는 필요**합니다.

### 3-A.3 "추가 IP를 쓰고 싶지 않다"면 — Multus가 아니라 다른 선택지

목표가 **"추가 IP 없이, 노드 IP로 SNAT되어 특정(NAS) NIC로 나가기"** 라면, 이는 **Multus의 문제가 아니라 "Pod egress를 호스트 네트워크 스택으로 태워 노드 IP로 SNAT"** 하는 경로의 문제입니다. 정리하면:

| 방식 | 추가 IP 사용 | S3 트래픽 소스 IP | NAS NIC 선택 방법 | hostNetwork | 비고 |
|------|:-----------:|-------------------|-------------------|:-----------:|------|
| **Multus (macvlan 등)** | **필요(보조 NIC IP)** | net1 IP(Pod 고유) | Pod netns 라우팅(net1) | false | 깔끔한 분리·격리. 단 IP 추가 필요 |
| **hostNetwork: true** | 불필요(노드 IP 사용) | **노드 IP(NAS NIC의 노드 IP)** | 노드 라우팅/정책 라우팅 | **true** | 추가 IP 없음. 단 Pod=노드 netns(격리 약화·포트충돌) |
| **OVN Local GW + 호스트 정책 라우팅 + ipForwarding:Global** | 불필요(노드 IP SNAT) | **노드 IP** | 호스트 커널 라우팅(S3→NAS NIC) | false | [`02-ipforwarding-global-egress.md`](02-ipforwarding-global-egress.md)·[`03-...`](03-nasnic-egress-and-routable-podcidr.md) 참조. OVN-K 한정 |
| **Antrea + 노드 SNAT + 정책 라우팅** | 불필요(노드 IP SNAT) | **노드 IP** | 호스트/노드 라우팅 | false | Antrea CNI 한정 |
| EgressIP | 추가 IP(고정 egress IP) | 지정 egress IP | OVN egress 노드 | false | 사용자가 원치 않는 방식 |

- 즉 **"추가 IP를 안 쓰고 노드 IP SNAT"** 를 원하면:
  1. **hostNetwork:true** (가장 단순, 추가 IP 0). 단 "내부 pod 호출 + NAS S3" 동시 워크로드에서 격리/충돌 이슈 가능(이전 문서들의 딜레마).
  2. **CNI egress 경로 + 호스트 정책 라우팅**(OVN Local GW + Global, 또는 Antrea 노드 SNAT). Pod egress가 **호스트 커널 라우팅을 경유**하도록 만들고, 호스트에 "S3→NAS NIC" 경로를 넣으면 **노드 IP로 SNAT되어 NAS NIC로** 나갑니다. 이 경우 **보조 IP가 필요 없습니다.**

### 3-A.4 그래서 무엇을 선택할까 (권고)

- **추가 IP를 쓰지 않는 것이 최우선**이고, S3 트래픽 소스가 **노드 IP여도 무방**하다면 → **Multus를 쓰지 말고**, "CNI egress를 호스트 라우팅으로 태워 노드 IP SNAT"( OVN Local GW+`ipForwarding:Global`+정책 라우팅, 또는 Antrea 노드 SNAT+정책 라우팅 )를 검토하세요. 추가 IP가 0입니다.
- 반대로 **S3/NAS 쪽에서 "출발지를 Pod별 고유 IP로 식별/ACL"** 해야 하거나, 트래픽을 물리적으로 깔끔히 분리하고 싶다면 → **Multus(macvlan) + 보조 IP** 가 맞습니다(IP 1개/Pod 추가).
- 요약: **"추가 IP 회피 = 호스트 egress 경로(노드 IP SNAT)"**, **"트래픽 분리/고유 소스 IP = Multus(IP 필요)"**. 두 목적은 상충하므로 우선순위를 정해야 합니다.

> 즉, 질문에 대한 직접적 답: **Multus를 쓰는 한 보조 NIC 고유 IP는 필요**하고, **추가 IP 없이 노드 IP SNAT로 특정 NIC egress**를 원하면 그건 Multus가 아니라 **hostNetwork:false 유지 시 "CNI egress + 호스트 정책 라우팅 + (OVN이면)ipForwarding:Global"**, 또는 가장 단순하게는 **hostNetwork:true** 로 접근해야 합니다.

---

## 4. 패킷 분석

### 4.1 인터페이스/라우팅 상태 (Pod netns)

```bash
# Pod netns 인터페이스
$ ip -br addr
eth0   UP  10.96.1.102/23         # 기본 CNI
net1   UP  192.168.90.51/24       # macvlan (NAS)

# Pod netns 라우팅
$ ip route
default via 10.96.0.1 dev eth0                 # 기본 경로: 클러스터
10.96.0.0/16 dev eth0                          # 클러스터 대역
192.168.90.0/24 dev net1                       # ★ S3(NAS) 대역 → net1
```

### 4.2 시나리오별 패킷 흐름

**(a) Pod → 클러스터 내부 service/pod (`10.96.x` / ClusterIP)**

```
[Pod netns] 라우팅 매칭: default/10.96.0.0/16 → eth0
[eth0]      src 10.96.1.102 → dst 10.96.8.8:8080   (기존 CNI 처리, SNAT 없음)
```

**(b) Pod → S3 (NAS, `192.168.90.20:443`)**

```
[Pod netns] 라우팅 매칭: 192.168.90.0/24 → net1
[net1 (macvlan)] src 192.168.90.51 → dst 192.168.90.20:443   (고유 MAC, NAS망 직접)
[노드 ens224]    동일 프레임이 물리 NAS NIC로 송출 (SNAT 아님, macvlan IP 그대로)
[S3 서버]        수신 시 소스 = 192.168.90.51 (Pod의 net1 IP)
```

**(c) 외부 인바운드 → Pod:8080 (NodePort/NPL 등)**

```
[eth0 경로]  기존 CNI/NPL로 도달 (net1 무관)
```

> 핵심 관찰: **(a)와 (b)가 서로 다른 인터페이스로 동시에** 성립합니다. (b)의 소스 IP는 노드 IP가 아니라 **net1의 macvlan IP**입니다. 만약 §2.3처럼 default route를 net1으로 옮기면 (a)가 깨져 "동시 성립"이 무너집니다(그래서 대역별 라우트만 추가하는 §2.1 권장).

### 4.3 tcpdump 관측 지점

```bash
# 1) Pod net1 에서 S3 트래픽
oc exec app-with-nas -- tcpdump -ni net1 host 192.168.90.20
#   기대: src 192.168.90.51 → dst 192.168.90.20:443

# 2) 노드 NAS 물리 NIC 에서
tcpdump -ni ens224 host 192.168.90.20
#   기대: src 192.168.90.51 (macvlan IP 그대로, SNAT 아님)

# 3) Pod eth0 에서 클러스터 내부 트래픽 (동시 확인)
oc exec app-with-nas -- tcpdump -ni eth0 host 10.96.8.8
```

### 4.4 자주 겪는 이슈 (macvlan 특성)

- **macvlan hairpin 제약:** macvlan은 기본적으로 **같은 물리 NIC의 노드 자신(호스트)과는 직접 통신 불가**(별도 우회 필요). S3가 노드 자신이 아니면 무관.
- **응답 대칭성:** S3/NAS 쪽에서 `192.168.90.51`로 돌아오는 경로가 성립해야 함(같은 L2면 자연 성립).
- **IPAM 충돌:** static IP는 NAS 세그먼트에서 유일해야 함. DHCP 사용 시 macvlan+dhcp IPAM 데몬 고려.
- **NetworkPolicy:** 표준 K8s NetworkPolicy는 기본망(eth0) 위주. 보조망 정책은 MultiNetworkPolicy 등 별도.

---

## 5. 요약

| 항목 | 내용 |
|------|------|
| Multus란 | 여러 CNI를 조합해 Pod에 복수 인터페이스(net1..)를 붙이는 메타 CNI |
| 핵심 오브젝트 | NetworkAttachmentDefinition(NAD) — 보조 인터페이스 생성 방식 정의 |
| IP별 NIC 라우팅 원리 | NAD로 붙인 net1의 **Pod netns 라우팅 테이블**이 목적지(S3 대역)에 따라 인터페이스를 선택 |
| hostNetwork:false 가능? | **가능.** 기본망(eth0)은 그대로, S3 대역만 net1으로. default route는 옮기지 않는 것이 안전 |
| S3 트래픽 소스 IP | net1(macvlan) IP 그대로(노드 IP SNAT 아님) |
| 주의 | default route override 시 클러스터 통신 단절 위험, macvlan hairpin/응답 대칭성/IPAM 충돌 |

> 결론: "노드에 multi NIC 라우팅이 있고, hostNetwork:false로 IP별 NIC 라우팅"을 하려면 **Multus 보조 네트워크(예 macvlan) + NAD의 대역별 route** 가 정공법입니다. Pod netns 라우팅이 목적지별로 인터페이스를 나눠, 내부 통신(eth0)과 NAS S3 통신(net1)을 동시에 성립시킵니다. F5-CIS 두 모드에서의 처리는 각 아키텍처 문서( [ClusterIP](../f5-cis-integration/01-f5-cis-clusterip-ovn-architecture.md) / [NodePortLocal](../f5-cis-integration/02-f5-cis-nodeportlocal-antrea-architecture.md) )의 "Multus 보조 네트워크" 섹션 참조.

---

## 6. 출처 (링크 검증 완료)

| 출처 | 링크 | 상태 |
|------|------|------|
| OKD: Understanding multiple networks | https://docs.okd.io/latest/networking/multiple_networks/understanding-multiple-networks.html | 200 OK |
| OKD: Attaching a pod to a secondary network | https://docs.okd.io/latest/networking/multiple_networks/secondary_networks/attaching-pod.html | 200 OK |
| openshift/multus-cni: macvlan-pod.yml 예시 | https://raw.githubusercontent.com/openshift/multus-cni/main/examples/macvlan-pod.yml | 200 OK |
| k8snetworkplumbingwg/multus-cni Issue #847 (default-route 영향) | https://github.com/k8snetworkplumbingwg/multus-cni/issues/847 | 200 OK |
| Red Hat ODF: Creating Multus networks (macvlan) | https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.18/html/managing_and_allocating_storage_resources/creating-multus-networks_rhodf | 페이지 유효(스크립트 403 봇차단) |
| Red Hat OCP 4.20: Multiple networks | https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/multiple_networks/index | 페이지 유효(스크립트 403 봇차단) |
| Red Hat Blog: Using the Multus CNI in OpenShift | https://www.redhat.com/en/blog/using-the-multus-cni-in-openshift | 페이지 유효(스크립트 403 봇차단) |
| Red Hat: MicroShift Multiple networks (MACVLAN 고유 MAC) | https://docs.redhat.com/en/documentation/red_hat_build_of_microshift/4.22/html/networking/multiple-networks | 페이지 유효(스크립트 403 봇차단) |
| Red Hat ODF: Multus network configuration (IPAM/whereabouts) | https://docs.redhat.com/ja/documentation/red_hat_openshift_data_foundation/4.20/html/red_hat_openshift_data_foundation_architecture/multus-network-configuration_mcg | 페이지 유효(스크립트 403 봇차단) |
| Antrea: Multus cookbook | https://antrea.io/docs/main/docs/cookbooks/multus/ | 200 OK |

> docs.redhat.com / redhat.com 블로그는 자동화 클라이언트에 403(봇 차단)이나 브라우저/조회도구로는 정상(끊어진 링크 아님). 무봇차단 미러로 OKD/GitHub 링크를 병행 제시했습니다.
