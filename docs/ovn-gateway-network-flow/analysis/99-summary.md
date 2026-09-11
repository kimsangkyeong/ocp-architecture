# 분석 - 최종 요약 (시나리오 × 비교안)

## 변경이력 (Change History)

| 버전 | 일자 | 작성자 | 작성/검증 | 변경 내용 |
|------|------------|-------------|-----------|-----------|
| v1.0 | 2026-09-11 | k.s.k & kiro | 1차 생성 + 자체 교차검증 | 시나리오/비교안 기준 요약 매트릭스 최초 작성 |

> 상세 근거는 시나리오별 분석 문서 및 [`../references/`](../references/) 참조.

---

## 1. 비교안 정의 (재게시)

| 비교안 | `routingViaHost` | `ipForwarding` | Gateway Mode |
|--------|------------------|----------------|--------------|
| 1안 | `true` | `Global` | Local GW |
| 2안 | `true` | `Restricted` | Local GW |
| 3안 | `false` | `Global` | Shared GW |
| 4안 | `false` | `Restricted` | Shared GW (**OCP 기본**) |

---

## 2. 통신 가능 여부 매트릭스 (시나리오 × 비교안)

| 시나리오 | 트래픽 | 1안 | 2안 | 3안 | 4안 |
|----------|--------|:---:|:---:|:---:|:---:|
| 시나리오1 : Pod A → Pod B (동일 노드, 8080) | East-West (노드 내부) | O | O | O | O |
| 시나리오2 : Pod A → Pod X (노드 간, 8080) | East-West (GENEVE) | O | O | O | O |
| 시나리오3 : Pod A → 외부 C (443) | North-South egress | O | O | O | O |

> 설계요구사항의 3개 시나리오는 **모두 OVN-Kubernetes가 관리하는 트래픽**(East-West 오버레이, 표준 egress SNAT)이므로, `routingViaHost`/`ipForwarding` 4개 조합 어디에서도 **통신 가능(O)** 합니다. 두 파라미터는 통신 "가능 여부"가 아니라 **패킷이 지나가는 경로(datapath)** 와 **노드의 일반 라우터 역할 범위**를 바꿉니다.

---

## 3. 소스/타겟 IP 요약 매트릭스

| 시나리오 | 관측 지점 | 소스 IP | 타겟 IP | 포트 | 비고 |
|----------|-----------|---------|---------|------|------|
| 1 | Pod A / node1 / Pod B | `10.96.1.102` | `10.96.8.8` | 8080 | SNAT 없음, br-int 내부 스위칭 |
| 2 | Pod A / Pod X (inner) | `10.96.1.102` | `10.96.9.9` | 8080 | inner 원본 보존 |
| 2 | node1↔node3 (outer) | `10.187.0.3` | `10.187.0.10` | UDP 6081 | GENEVE outer |
| 3 | Pod A (SNAT 전) | `10.96.1.102` | `192.168.56.7` | 443 | egress 진입 |
| 3 | node1 egress / 외부 C 수신 | `10.187.0.3` | `192.168.56.7` | 443 | 노드 IP로 SNAT |

> 위 IP:port 값은 4개 비교안에서 **동일**합니다(egress datapath만 상이). 시나리오3에서 외부 서버 C가 보는 소스 IP는 항상 노드 IP `10.187.0.3` 입니다(EgressIP 미적용 가정).

---

## 4. 비교안별 특성 요약 (datapath · 운영 관점)

| 비교안 | egress datapath | 호스트 커널 경유 | 성능/offload | 호스트 라우팅/방화벽 적용 | 노드 일반 포워딩 | 권장/주의 |
|--------|-----------------|:----------------:|:------------:|:--------------------------:|:----------------:|-----------|
| 1안 (Local, Global) | mp0→호스트→NIC | 예 | 상대적 낮음 | 예 (호스트 iptables/route) | 전역 ON | 호스트 라우팅 제어가 필요하거나, 노드가 라우터 역할을 해야 할 때 |
| 2안 (Local, Restricted) | mp0→호스트→NIC | 예 | 상대적 낮음 | 예 | OVN 한정 | Local GW가 필요하지만 노드를 일반 라우터로 쓰지 않을 때 |
| 3안 (Shared, Global) | OVS(br-ex) 직접 | 아니오 | 높음 | 아니오(OVS 처리) | 전역 ON | Shared 성능 + 노드 포워딩 동시 필요 시(드묾). Global 남용 주의 |
| 4안 (Shared, Restricted) | OVS(br-ex) 직접 | 아니오 | 높음 | 아니오 | OVN 한정 | **OCP 기본. 대부분의 표준 배포 권장값** |

---

## 5. F5 (CIS) 연동 관점 핵심 시사점

- 설계요구사항의 3개 시나리오 자체는 4개 안 모두 O 이지만, **F5 BIG-IP ↔ Pod 직접 라우팅(static route / no-tunnel)** 토폴로지에서는 `ipForwarding`가 실질 변수로 작용할 수 있습니다.
  - CIS는 노드 subnet을 BIG-IP의 정적 경로로 등록하여 터널 없이 Pod로 직접 라우팅합니다. 원문 내용을 재구성함. [F5 clouddocs: Static Route Support](https://clouddocs.f5.com/containers/latest/userguide/static-route-support.html)
  - iCNIv1(hybrid overlay) 방식은 OpenShift 4.13+ 에서 제거되어, OVN-Kubernetes + static route 방식이 권장됩니다. 원문 내용을 재구성함. [F5 clouddocs: OpenShift 4.8 & CIS OVN-Kubernetes](https://clouddocs.f5.com/containers/latest/userguide/openshift/openshift-4-8-cluster.html)
- 노드가 BIG-IP와 Pod 네트워크 사이의 **중계 라우터** 역할을 해야 하는 특정 반환경로(return path) 설계에서는 `ipForwarding: Global`이 필요할 수 있으므로, PoC에서 tcpdump로 양방향(요청/응답) 경로를 반드시 확인하십시오.
- Gateway mode 선택(Local/Shared)은 egress 소스 IP(노드 IP SNAT) 결과를 바꾸지 않으나, **호스트 방화벽/라우팅 정책의 적용 여부**(Local=적용, Shared=우회)가 달라지므로 BIG-IP 헬스체크/SNAT 풀 설계 시 함께 고려해야 합니다.

> Content was rephrased for compliance with licensing restrictions.

---

## 6. 종합 결론

1. **통신 가능 여부:** 3개 시나리오 × 4개 비교안 = 12개 조합 모두 **O(통신 가능)**. (표준 OVN-K 관리 트래픽이므로)
2. **소스/타겟 IP:** East-West는 Pod IP 보존(SNAT 없음), 외부 egress는 노드 IP로 SNAT — 4개 안 동일.
3. **두 파라미터의 실제 의미:**
   - `routingViaHost` = **egress 경로**(Local: 호스트 커널 경유 / Shared: OVS 직접). 통신 가부는 불변, 성능·호스트정책 적용 여부가 상이.
   - `ipForwarding` = **노드의 비 OVN 라우터 역할 범위**. 표준 Pod 통신엔 무영향(Restricted 충분). 노드를 라우터로 쓰는 토폴로지(F5 static-route 등)에서만 Global 필요성 검토.
4. **권장:** 특별한 요구(호스트 라우팅 제어, 노드 라우터 역할)가 없으면 **4안(Shared + Restricted, OCP 기본)** 이 표준.
