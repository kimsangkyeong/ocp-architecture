# 참고문서 01 - OVN-Kubernetes Gateway 개념 (`routingViaHost` / `ipForwarding`)

## 변경이력 (Change History)

| 버전 | 일자 | 작성자 | 작성/검증 | 변경 내용 |
|------|------------|-------------|-----------|-----------|
| v1.0 | 2026-09-11 | k.s.k & kiro | 1차 생성 + 자체 교차검증 | OCP 4.20+ 기준 `gatewayConfig`의 `routingViaHost` / `ipForwarding` 개념 최초 작성 |

> 참고: 본 문서는 상세 분석 문서( `../analysis/*.md` )가 반복 참조하는 **개념 레퍼런스**입니다. 시나리오별 패킷 분석은 분석 문서를 참조하세요.

---

## 1. 대상 오브젝트: `Network.operator.openshift.io` 의 `gatewayConfig`

OCP 4.20+ 에서 두 파라미터는 Cluster Network Operator(CNO)가 관리하는 `Network` 오브젝트의 `spec.defaultNetwork.ovnKubernetesConfig.gatewayConfig` 하위에 위치합니다.

```yaml
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  clusterNetwork:
    - cidr: 10.96.0.0/16      # (본 문서 전제 - Pod pool)
      hostPrefix: 23
  serviceNetwork:
    - 10.128.0.0/16           # (본 문서 전제 - Service pool)
  defaultNetwork:
    type: OVNKubernetes
    ovnKubernetesConfig:
      gatewayConfig:
        routingViaHost: false     # true=Local GW,  false=Shared GW(기본)
        ipForwarding: Restricted  # Restricted(기본) | Global
```

> 주의: 위 예시의 `clusterNetwork`(Pod) / `serviceNetwork`(Service) 값은 **본 분석의 전제 환경**(설계요구사항3)을 반영한 것입니다. OCP 기본 설치값과 다릅니다.

---

## 2. `routingViaHost` — Gateway Mode (Local vs Shared)

`routingViaHost`는 **Pod에서 나가는(egress, north-south) 트래픽이 노드의 호스트 커널 라우팅을 경유하는지** 를 결정합니다.

| 값 | Gateway Mode | 동작 요약 |
|----|--------------|-----------|
| `false` (기본) | **Shared Gateway** | egress 트래픽이 OVS 데이터패스 내부에 머무르며 `br-ex`(breth0)를 통해 **직접** 물리 NIC로 나감. 호스트 커널 라우팅 테이블을 **경유하지 않음**. |
| `true` | **Local Gateway** | egress 트래픽이 관리 포트(`ovn-k8s-mp0`)를 통해 **호스트 커널로 빠져나온 뒤** 호스트 라우팅 테이블/iptables를 경유해서 나감. |

- 근거(개념): OVN-Kubernetes 설계 문서 — Shared gateway는 트래픽이 OVS 데이터패스에 머물러 `breth0`로 직접 나가고, Local gateway는 관리 포트(`ovn-k8s-mp0`)를 통해 호스트 커널로 나간 뒤 호스트가 라우팅함. 원문 내용을 라이선스 준수를 위해 재구성함. [OVN-Kubernetes: External Bridge (breth0) Flows](https://ovn-kubernetes.io/master/design/bridge-flows/)
- 근거(OCP/OKD 문서): `true`는 egress 트래픽이 노드의 로컬 gateway를 통해 라우팅되어 **호스트 라우팅 테이블이 적용**되고, `false`는 트래픽이 호스트를 경유하지 않고 OVS가 노드 IP 인터페이스로 직접 출력함. 원문 내용을 재구성함. [OKD: Configuring a gateway](https://docs.okd.io/4.19/networking/ovn_kubernetes_network_provider/configuring-gateway.html)
- 근거(성능/특성): Shared gateway 모드에서는 ingress/egress 트래픽이 커널 라우팅 테이블을 우회하여 OVS를 통해 NIC로 직접 나가며, 성능/HW offload 이점 및 SNAT 감소가 있음. 원문 내용을 재구성함. [Red Hat Bugzilla 2042516 (Full Text)](https://bugzilla.redhat.com/show_bug.cgi?format=multiple&id=2042516)

> Content was rephrased for compliance with licensing restrictions.

### 2.1 핵심: East-West(Pod ↔ Pod)는 gateway mode와 무관

`routingViaHost`는 **north-south(외부로 나가는) egress 경로**를 제어합니다. **동일 클러스터 내부의 Pod ↔ Pod(East-West)** 통신은 gateway mode와 관계없이 **GENEVE 오버레이**를 통해 전달되며, 이 경로는 항상 OVN 논리 네트워크 안에서 처리됩니다.

- 근거: F5 CIS 문서 — OVN-Kubernetes는 클러스터 내부 EAST/WEST 트래픽에 GENEVE 프로토콜을 사용함. 원문 내용을 재구성함. [F5 clouddocs: OpenShift 4.8 & CIS OVN-Kubernetes](https://clouddocs.f5.com/containers/latest/userguide/openshift/openshift-4-8-cluster.html)

---

## 3. `ipForwarding` — 호스트의 라우터 역할 범위 (Restricted vs Global)

`ipForwarding`는 **노드(호스트)가 OVN-Kubernetes가 관리하지 않는(비 OVN) IP 트래픽을 라우팅(포워딩)해 줄 것인지** 를 결정합니다.

| 값 | 동작 요약 |
|----|-----------|
| `Restricted` (기본) | Kubernetes 관련 트래픽(Pod/Service/masquerade)은 정상 포워딩되지만, **그 외의 IP 트래픽은 노드가 라우팅하지 않음**. 포워딩 sysctl이 OVN-Kubernetes 자체 인터페이스(관리 포트 + 브리지)에만 활성화됨. |
| `Global` | 호스트가 **모든** 트래픽을 포워딩. 즉 `net.ipv4.ip_forward=1`(전역)로 노드가 일반 라우터처럼 동작하여 OVN이 관리하지 않는 트래픽도 인터페이스 간 포워딩함. |

- 근거(정의): 기본값은 Restricted이며 Kubernetes 관련 트래픽은 계속 포워딩되지만 그 외 IP 트래픽은 노드가 라우팅하지 않음. 호스트가 OVN-Kubernetes 인터페이스를 통해 트래픽을 포워딩하게 하려면 Global로 설정. 지원값은 Restricted/Global. 원문 내용을 재구성함. [Red Hat Solution 7053694](https://access.redhat.com/solutions/7053694)
- 근거(구현 메커니즘): IPv4 활성 시 OVN-Kubernetes는 관리 포트/브리지 인터페이스에만 `net.ipv4.conf.[IFNAME].forwarding=1`을 설정하여 해당 인터페이스로만 포워딩을 허용함(관리자가 전역 `net.ipv4.ip_forward`를 직접 켜지 않는 한). 원문 내용을 재구성함. [OVN-Kubernetes Config Variables (configuration.md)](https://raw.githubusercontent.com/ovn-kubernetes/ovn-kubernetes/master/docs/getting-started/configuration.md)
- 근거(활성화 관리): CNO를 통해 IP forwarding 활성화 등 네트워킹을 관리할 수 있음. 원문 내용을 재구성함. [Red Hat OCP 4.20: Cluster Network Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/networking_operators/cluster-network-operator)

> Content was rephrased for compliance with licensing restrictions.

### 3.1 매우 중요한 구분 — `ipForwarding`가 영향을 주는 트래픽

`ipForwarding`는 **노드가 "제3자 트래픽"의 중계 라우터로 동작해야 하는 경우에만** 영향을 줍니다. 다음을 명확히 구분해야 합니다.

- **영향 없음(Restricted에서도 정상):**
  - Pod → Pod (East-West, GENEVE 오버레이)
  - Pod → ClusterIP Service
  - Pod → 외부(egress). Pod 트래픽이 노드 IP로 SNAT(masquerade)되어 나가는 것은 OVN-Kubernetes가 관리하는 트래픽이므로 Restricted에서도 동작.
- **영향 있음(Global이 필요할 수 있음):**
  - 노드가 **자신이 종단이 아닌** 트래픽을, 예를 들어 **보조 인터페이스(secondary NIC) ↔ Pod 네트워크** 사이에서 라우팅해 주어야 하는 경우.
  - `routingViaHost: true`(Local GW) + 보조 인터페이스(br-ex를 별도 NIC로) 조합에서 host-network Pod가 기본 kubernetes service IP로 라우팅되지 못하는 케이스 등. 원문 내용을 재구성함. [Red Hat Solution 7053694](https://access.redhat.com/solutions/7053694)
  - F5 BIG-IP ↔ Pod 직접 라우팅(static route, no-tunnel)에서 노드가 return 경로의 중계자 역할을 해야 하는 특정 토폴로지.

> 정리: 본 분석의 3개 시나리오(Pod→Pod, Pod→Pod cross-node, Pod→외부 egress)는 **모두 OVN-Kubernetes가 관리하는 트래픽**이므로, 통신 가능 여부 자체는 `ipForwarding`(Restricted/Global) 값에 의해 바뀌지 않습니다. `ipForwarding`는 "노드를 일반 라우터로 사용하는" 별도 토폴로지에서 의미를 갖습니다. 이 점은 분석 문서에서 시나리오별로 O/X와 함께 명시합니다.

---

## 4. 두 파라미터 조합 (비교 1안 ~ 4안)

| 비교안 | `routingViaHost` | `ipForwarding` | Gateway Mode | 호스트 일반 라우팅 |
|--------|------------------|----------------|--------------|--------------------|
| 1안 | `true` | `Global` | Local GW | 전역 포워딩 ON |
| 2안 | `true` | `Restricted` | Local GW | OVN 인터페이스 한정 |
| 3안 | `false` | `Global` | Shared GW | 전역 포워딩 ON |
| 4안 | `false` | `Restricted` | Shared GW | OVN 인터페이스 한정 (**OCP 기본 조합**) |

---

## 5. SNAT / Masquerade 개념 (egress 시 소스 IP 변환)

- **ClusterIP / East-West Pod 통신:** 클러스터 내부에서 ClusterIP로 가는 패킷은 (iptables 모드 기준) **SNAT 되지 않아 소스 Pod IP가 보존**됩니다. 원문 내용을 재구성함. [Kubernetes: Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/)
- **외부(north-south) egress:** Pod가 외부로 나갈 때는 소스 Pod IP가 **노드 IP로 SNAT(masquerade)** 됩니다. OVN-Kubernetes는 이 과정에서 외부 브리지(breth0) 상의 중간 SNAT 주소로 **masquerade IP**(물리망에 노출되면 안 되는 노드 로컬 합성 주소)를 사용합니다. 원문 내용을 재구성함. [OVN-Kubernetes: Masquerade IPs](https://ovn-kubernetes.io/master/design/masquerade-ips/)
- **일관된 egress 소스 IP가 필요할 때:** EgressIP 기능으로 특정 네임스페이스/Pod의 외부 통신 소스 IP를 고정할 수 있습니다. 원문 내용을 재구성함. [OVN-Kubernetes: Egress IP](https://ovn-kubernetes.io/master/features/cluster-egress-controls/egress-ip/)

> Content was rephrased for compliance with licensing restrictions.

---

## 6. 참조

전체 출처 목록 및 링크 검증 결과는 [`03-glossary-and-sources.md`](03-glossary-and-sources.md) 를 참조하세요.
