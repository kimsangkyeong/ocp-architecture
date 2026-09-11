# OCP 4.20+ OVN-Kubernetes Gateway 네트워크 흐름 분석

`routingViaHost` / `ipForwarding` 설정(비교 1~4안)에 따른 Pod 통신 패킷 흐름 분석 문서 모음입니다.

## 변경이력 (Change History)

| 버전 | 일자 | 작성자 | 작성/검증 | 변경 내용 |
|------|------------|-------------|-----------|-----------|
| v1.0 | 2026-09-11 | k.s.k & kiro | 1차 생성 + 자체 교차검증 + 링크 검증 | 문서 세트 최초 구성(참고문서 3 + 분석 4 + 인덱스) |
| v1.1 | 2026-09-11 | k.s.k & kiro | 심화 추가 + 링크 검증 | 심화 문서 2건 추가(deep-dive): br-int/다중 NIC egress, ipForwarding Global egress 효과(Q1~Q3) |
| v1.2 | 2026-09-11 | k.s.k & kiro | F5 연동 아키텍처 추가 + 링크 검증 | F5 CIS(ClusterIP mode) ↔ OCP(OVN-K) 연동 아키텍처 문서 추가(f5-cis-integration): static route/no-tunnel, SNAT, ipForwarding/routingViaHost 관계 |
| v1.3 | 2026-09-11 | k.s.k & kiro | 변경이력 컬럼 추가 | 전체 문서 변경이력 표에 "작성자" 컬럼 추가(값: k.s.k & kiro), 이후 변경이력에도 동일 적용 |
| v1.4 | 2026-09-11 | k.s.k & kiro | 심화 추가 + 링크 검증 | 심화 03 추가(deep-dive): NAS NIC egress(hostNetwork:false) 가능성, Pod CIDR 실IP대역(Route Advertisements/No-Overlay BGP) 구성 |
| v1.5 | 2026-09-11 | k.s.k & kiro | F5 NPL+Antrea 아키텍처 추가 + 링크 검증 | F5 CIS(NodePortLocal) ↔ OCP(Antrea CNI) 네트워크 구조 문서 추가(f5-cis-integration): NPL 동작, scenario1~3 설정권고, 동일Pod 내부+NPL+NAS S3, overlay/underlay Pod CIDR 비교 |
| v1.6 | 2026-09-11 | k.s.k & kiro | Multus 심화 추가 + 아키텍처 문서 보강 | 심화 04(Multus 보조 네트워크: 원리/NAD 구성/패킷 분석) 추가, F5 ClusterIP(5-A)·NodePortLocal(4-A) 문서에 Multus 처리 섹션 추가 |
| v1.7 | 2026-09-11 | k.s.k & kiro | Multus IP 요건 Q&A 추가 | 심화 04에 3-A절 추가: 보조 NIC 고유 IP 필요 여부 및 "추가 IP 없이 노드 IP SNAT" 대안(hostNetwork / CNI egress+정책라우팅) 비교 |
| v1.8 | 2026-09-11 | k.s.k & kiro | NodePort listen 섹션 추가 | F5 ClusterIP(5-B)·NodePortLocal(4-B) 문서에 hostNetwork 없이 listen(0.0.0.0,30000) 가능 여부 및 올바른 구성(targetPort+Service/NPL) 추가 |
| v1.9 | 2026-09-11 | k.s.k & kiro | TCP/UDP NodePort 소켓 심화 추가 | 심화 05 추가(deep-dive): TCP/UDP 소켓의 NodePort Service 동작(SNAT/externalTrafficPolicy/UDP 응답), 내부 호출 병행, 앱이 노드포트 직접 소유(hostNetwork/hostPort) 사례 |

---

## 문서 구성 (기본요구사항5 - 참조 분리)

가독성을 위해 개념/환경/출처는 **참고문서**로 분리하고, 시나리오별 분석 문서가 이를 참조합니다.

```
docs/ovn-gateway-network-flow/
├── README.md                          ← (현재 문서) 인덱스 + 요구사항 처리 노트
├── references/                        ← 참고문서 (반복 인용되는 개념/환경/출처)
│   ├── 01-ovn-gateway-concepts.md     ← routingViaHost / ipForwarding 개념, 비교안 정의
│   ├── 02-cluster-environment.md      ← 클러스터/노드/Pod/외부서버 환경, tcpdump 관측지점
│   └── 03-glossary-and-sources.md     ← 용어, 출처 링크 검증 결과, 품질보증 노트
└── analysis/                          ← 시나리오별 상세 분석 (구성도 + 패킷 + O/X)
    ├── scenario-1-podA-to-podB.md     ← 시나리오1: 동일 노드 East-West (8080)
    ├── scenario-2-podA-to-podX.md     ← 시나리오2: 노드 간 East-West / GENEVE (8080)
    ├── scenario-3-podA-to-external.md ← 시나리오3: 외부 egress / SNAT (443)
    └── 99-summary.md                  ← 시나리오 × 비교안 최종 요약 매트릭스
├── deep-dive/                         ← 심화 Q&A (실제 운영 이슈 상세)
│   ├── 01-brint-and-multinic-egress.md    ← Q1 br-int 의미 / Q2 다중 NIC egress·hostNetwork
│   ├── 02-ipforwarding-global-egress.md   ← Q3 F5-CIS·hostNetwork 충돌 / ipForwarding Global 효과
│   ├── 03-nasnic-egress-and-routable-podcidr.md ← NAS NIC egress(hostNetwork:false) / Pod CIDR 실IP대역(BGP)
│   ├── 04-multus-secondary-network.md    ← Multus 보조 네트워크 원리/NAD 구성/패킷 분석
│   └── 05-nodeport-socket-tcp-udp.md     ← TCP/UDP 소켓 NodePort 동작 / 앱이 노드포트 직접 소유(hostNetwork/hostPort)
└── f5-cis-integration/                ← F5 연동 아키텍처 (Network 관점)
    ├── 01-f5-cis-clusterip-ovn-architecture.md      ← F5 CIS(ClusterIP) ↔ OCP(OVN-K) static route/SNAT/ipForwarding
    └── 02-f5-cis-nodeportlocal-antrea-architecture.md ← F5 CIS(NodePortLocal) ↔ OCP(Antrea CNI) NPL/overlay·underlay
```

## 빠른 링크

- 개념: [routingViaHost / ipForwarding](references/01-ovn-gateway-concepts.md)
- 환경/관측지점: [클러스터 환경](references/02-cluster-environment.md)
- 용어/출처: [Glossary & Sources](references/03-glossary-and-sources.md)
- 시나리오1: [Pod A → Pod B](analysis/scenario-1-podA-to-podB.md)
- 시나리오2: [Pod A → Pod X](analysis/scenario-2-podA-to-podX.md)
- 시나리오3: [Pod A → 외부 서버 C](analysis/scenario-3-podA-to-external.md)
- **최종 요약: [Summary](analysis/99-summary.md)**
- 심화 Q1·Q2: [br-int 의미 / 다중 NIC egress·hostNetwork](deep-dive/01-brint-and-multinic-egress.md)
- 심화 Q3: [ipForwarding Global egress 효과 / F5-CIS·hostNetwork 충돌](deep-dive/02-ipforwarding-global-egress.md)
- 심화: [NAS NIC egress(hostNetwork:false) / Pod CIDR 실IP대역(BGP)](deep-dive/03-nasnic-egress-and-routable-podcidr.md)
- 심화: [Multus 보조 네트워크 (원리·NAD 구성·패킷 분석)](deep-dive/04-multus-secondary-network.md)
- 심화: [TCP/UDP 소켓 NodePort 동작 / 앱이 노드포트 직접 소유](deep-dive/05-nodeport-socket-tcp-udp.md)
- **F5 연동: [F5 CIS(ClusterIP) ↔ OCP(OVN-K) 아키텍처 (Network)](f5-cis-integration/01-f5-cis-clusterip-ovn-architecture.md)**
- **F5 연동: [F5 CIS(NodePortLocal) ↔ OCP(Antrea CNI) 아키텍처 (Network)](f5-cis-integration/02-f5-cis-nodeportlocal-antrea-architecture.md)**

---

## 핵심 결론 (한 줄 요약)

3개 시나리오 × 4개 비교안 = **12개 조합 모두 통신 가능(O)**. `routingViaHost`/`ipForwarding`는 통신 "가부"가 아니라 egress **경로(datapath)** 와 노드의 **비 OVN 라우터 역할 범위**를 바꿉니다. 특별 요구가 없으면 **4안(Shared + Restricted, OCP 기본)** 권장.

| 시나리오 | 1안 | 2안 | 3안 | 4안 |
|----------|:---:|:---:|:---:|:---:|
| 1 (동일 노드) | O | O | O | O |
| 2 (노드 간) | O | O | O | O |
| 3 (외부 egress) | O | O | O | O |

---

## 기본요구사항 처리 현황

| 요구사항 | 처리 |
|----------|------|
| 1. 폴더+md 저장 | `docs/ovn-gateway-network-flow/` 하위에 md로 구성 |
| 2. 이미지/다이어그램 | 비용 0의 **Mermaid** 다이어그램으로 구성도 표현(각 분석 문서). 유료 이미지 생성 미사용 |
| 3. 2개 모델 상호검증 | 단일 실행 모델 환경으로 2개 모델 물리 동시 실행 불가 → **문서화된 자체 교차검증**(독립 출처 대조)으로 대체. 상세: [03-glossary-and-sources.md](references/03-glossary-and-sources.md) §3 |
| 4. 비용 절감 | 유료 리소스 미사용(Mermaid + 무료 공개 출처). 유료 필요 시 사전 승인 절차 |
| 5. 참조 분리 | 참고문서(references) / 분석(analysis) 분리, 상호 링크 |
| 6. 변경이력 | 모든 문서 **맨 앞에 변경이력** 표 배치 |

> 요구사항3(2개 모델)·요구사항2(이미지)는 현재 실행 환경 제약으로 위와 같이 처리했습니다. (a) 특정 2차 모델로 교차 리뷰, (b) 유료 다이어그램/이미지 렌더링을 원하시면 방식과 비용을 안내한 뒤 승인받아 진행하겠습니다.

## 주요 출처 (링크 검증 완료)

전체 목록/검증 결과는 [references/03-glossary-and-sources.md](references/03-glossary-and-sources.md) 참조.

- [OVN-Kubernetes: bridge-flows](https://ovn-kubernetes.io/master/design/bridge-flows/)
- [OVN-Kubernetes: masquerade-ips](https://ovn-kubernetes.io/master/design/masquerade-ips/)
- [OVN-Kubernetes: Config Variables](https://raw.githubusercontent.com/ovn-kubernetes/ovn-kubernetes/master/docs/getting-started/configuration.md)
- [OKD: Configuring a gateway](https://docs.okd.io/4.19/networking/ovn_kubernetes_network_provider/configuring-gateway.html)
- [Red Hat Solution 7053694](https://access.redhat.com/solutions/7053694)
- [Kubernetes: Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/)
- [F5 clouddocs: OVN-Kubernetes CIS 가이드](https://clouddocs.f5.com/containers/latest/userguide/openshift/openshift-4-8-cluster.html)
- [F5 clouddocs: Static Route Support](https://clouddocs.f5.com/containers/latest/userguide/static-route-support.html)
