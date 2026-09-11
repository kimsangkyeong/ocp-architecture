# 참고문서 03 - 용어 및 출처(Cross-check) 목록

## 변경이력 (Change History)

| 버전 | 일자 | 작성자 | 작성/검증 | 변경 내용 |
|------|------------|-------------|-----------|-----------|
| v1.0 | 2026-09-11 | k.s.k & kiro | 1차 생성 + 링크 정상호출 검증 | 용어 정리 및 출처 링크(HTTP 상태) 검증 결과 최초 작성 |

---

## 1. 용어(Glossary)

| 용어 | 설명 |
|------|------|
| OVN-Kubernetes | Open Virtual Network(OVN) 기반 OCP 기본 CNI. 각 노드에서 OVS 실행, OVN이 OVS를 프로그래밍하여 선언된 네트워크 구성을 구현. |
| Gateway Mode | Pod egress(north-south)가 호스트 커널을 경유(Local)하는지, OVS로 직접(Shared) 나가는지 결정. `routingViaHost`로 설정. |
| `routingViaHost` | `true`=Local Gateway, `false`(기본)=Shared Gateway. |
| `ipForwarding` | 노드가 비 OVN 트래픽을 포워딩하는 범위. `Restricted`(기본)=OVN 인터페이스 한정, `Global`=전역 포워딩. |
| `br-ex` (breth0) | 노드 외부 브리지. 물리 NIC를 uplink로 편입하며 노드 물리 IP/MAC이 커널과 OVN 간 공유됨. |
| `ovn-k8s-mp0` | OVN-Kubernetes 관리 포트. Local GW에서 OVN↔호스트 커널 간 트래픽 통로. |
| GENEVE | OVN 오버레이 캡슐화 프로토콜(UDP 6081). 노드 간 East-West 트래픽 전송에 사용. |
| Masquerade IP | 외부 브리지 상에서 중간 SNAT 주소로 쓰이는 노드 로컬 합성 IP(물리망 비노출). |
| SNAT / Masquerade | egress 시 소스 IP를 노드 IP로 치환. 외부 통신에서 Pod IP → 노드 IP. |
| ClusterIP | 클러스터 내부에서만 유효한 Service 가상 IP. 내부 접근 시 소스 IP 보존(SNAT 없음, iptables 모드). |
| GR (Gateway Router) | OVN 논리 라우터. 외부 브리지 patch port를 통해 egress 처리. |
| CIS | F5 BIG-IP Container Ingress Services. OCP/K8s와 연동하여 BIG-IP에 L4/L7 서비스 동적 생성. |
| Static Route (F5 no-tunnel) | CIS가 노드 subnet을 BIG-IP 정적 경로로 구성하여 터널 없이 Pod로 직접 라우팅. |

---

## 2. 출처 및 링크 검증 결과

검증 방법: PowerShell `Invoke-WebRequest`(GET) 로 HTTP 상태 코드 확인 (2026-09-11 기준).

| # | 출처 | 링크 | 상태 | 비고 |
|---|------|------|------|------|
| 1 | OVN-Kubernetes: External Bridge (breth0) Flows | https://ovn-kubernetes.io/master/design/bridge-flows/ | 200 OK | Shared/Local GW 데이터패스 근거 |
| 2 | OVN-Kubernetes: Masquerade IPs | https://ovn-kubernetes.io/master/design/masquerade-ips/ | 200 OK | egress SNAT/masquerade IP 근거 |
| 3 | OVN-Kubernetes: Egress IP | https://ovn-kubernetes.io/master/features/cluster-egress-controls/egress-ip/ | 200 OK | 일관된 egress 소스 IP |
| 4 | OVN-Kubernetes: Config Variables (configuration.md) | https://raw.githubusercontent.com/ovn-kubernetes/ovn-kubernetes/master/docs/getting-started/configuration.md | 200 OK | ipForwarding sysctl/iptables 메커니즘 |
| 5 | OKD: Configuring a gateway | https://docs.okd.io/4.19/networking/ovn_kubernetes_network_provider/configuring-gateway.html | 200 OK | routingViaHost true/false 정의 |
| 6 | Red Hat Solution 7053694 | https://access.redhat.com/solutions/7053694 | 200 OK | ipForwarding Restricted/Global 정의 |
| 7 | Kubernetes: Using Source IP | https://kubernetes.io/docs/tutorials/services/source-ip/ | 200 OK | ClusterIP 내부 통신 소스 IP 보존 |
| 8 | F5 clouddocs: OpenShift 4.8 & CIS OVN-Kubernetes | https://clouddocs.f5.com/containers/latest/userguide/openshift/openshift-4-8-cluster.html | 200 OK | GENEVE E/W, iCNIv1 4.13+ 제거 |
| 9 | F5 clouddocs: Static Route Support | https://clouddocs.f5.com/containers/latest/userguide/static-route-support.html | 200 OK | BIG-IP no-tunnel 정적 경로 |
| 10 | F5 clouddocs: OVN-Kubernetes with BIG-IP HA no Tunnels (4.12) | https://clouddocs.f5.com/containers/latest/userguide/openshift/openshift-4-12-cluster.html | 200 OK | CIS cluster mode 최신 가이드 |
| 11 | Red Hat OCP 4.20: Cluster Network Operator | https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/networking_operators/cluster-network-operator | 페이지 유효 (스크립트 GET 시 403: 봇 차단) | 브라우저 정상 접근. CNO IP forwarding 관리 |
| 12 | Red Hat Bugzilla 2042516 (Full Text) | https://bugzilla.redhat.com/show_bug.cgi?format=multiple&id=2042516 | 페이지 유효 | Shared GW 커널 라우팅 우회 특성 |

> #11(docs.redhat.com)은 자동화 클라이언트(User-Agent 기반 봇 차단)에는 403을 반환하지만, 브라우저 및 본 작업의 검색/본문 조회 도구로는 정상 조회됩니다. 즉 **끊어진 링크가 아니라 봇 차단**입니다. 동일 내용의 무봇차단 미러로 OKD(#5) 및 상위 근거(#1~#6)를 병행 제시했습니다.

---

## 3. 자료 품질 보증 노트 (기본요구사항3, 4)

### 기본요구사항3 (2개 모델 상호검증)에 대한 처리
- 본 세션은 단일 실행 모델(서버 동적 선택, "Auto")로 동작하여 **서로 다른 2개 모델을 물리적으로 동시 실행할 수 없습니다.** 사실과 다른 방식(2개 모델 사용)을 주장하지 않습니다.
- 대신 다음의 **문서화된 자체 교차검증(self cross-validation)** 을 수행했습니다.
  1. 1차: 개념/동작 초안 작성.
  2. 2차: 상충 가능한 독립 출처(업스트림 OVN-K 설계 문서 ↔ Red Hat/OKD 제품 문서 ↔ 커널 sysctl 메커니즘)로 각 진술을 대조.
  3. 링크 HTTP 상태 실측(위 표) 후 정상 링크만 인용.
- 사용자가 별도의 2차 모델 검토를 원하시면, 본 문서를 다른 모델(예: 다른 LLM/리뷰어)에 넣어 리뷰하는 절차를 추가로 안내드릴 수 있습니다.

### 기본요구사항4 (비용 절감)에 대한 처리
- 유료 이미지 생성(다이어그램 렌더링 API 등)은 사용하지 않았습니다. 대신 **비용 0의 Mermaid 다이어그램**(md 내 렌더링)과 **공개 출처의 무료 이미지 링크**만 사용했습니다.
- 유료 리소스가 필요한 작업이 생기면, 진행 전에 대안과 비용을 먼저 제시하고 승인받는 절차를 따릅니다.
