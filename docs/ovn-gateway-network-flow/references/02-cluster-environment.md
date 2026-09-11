# 참고문서 02 - 테스트 클러스터 환경 및 토폴로지

## 변경이력 (Change History)

| 버전 | 일자 | 작성자 | 작성/검증 | 변경 내용 |
|------|------------|-------------|-----------|-----------|
| v1.0 | 2026-09-11 | k.s.k & kiro | 1차 생성 | 설계요구사항3 기준 클러스터/호출 환경 정보 최초 작성 |

---

## 1. OCP Cluster 네트워크 풀 (전제)

| 구분 | CIDR | 설명 |
|------|------|------|
| Service pool | `10.128.0.0/16` | ClusterIP 등 Service 가상 IP 대역 |
| Pod pool (clusterNetwork) | `10.96.0.0/16` | Pod IP 대역 (오버레이) |
| Machine pool | `10.187.0.0/24` | 노드(worker) 물리/머신 네트워크 대역 |

> 주의: 이 값들은 본 분석 전용 전제값입니다. OCP 기본 설치 기본값(예: Pod `10.128.0.0/14`, Service `172.30.0.0/16`)과 다릅니다.

## 2. 노드 / Pod / 외부 서버 정보

| 역할 | 노드 | 노드 IP (machine pool) | Pod | Pod IP (pod pool) |
|------|------|------------------------|-----|-------------------|
| 호출 소스 | worker node 1 | `10.187.0.3` | **Pod A** | `10.96.1.102` |
| 호출 타겟1 (동일 노드) | worker node 1 | `10.187.0.3` | **Pod B** | `10.96.8.8` |
| 호출 타겟1 (다른 노드) | worker node 3 | `10.187.0.10` | **Pod X** | `10.96.9.9` |
| 호출 타겟2 (외부) | 외부 서버 C | `192.168.56.7` | - | - |

## 3. 호출 시나리오

| 시나리오 | 소스 | 타겟 | 포트 | 트래픽 유형 |
|----------|------|------|------|-------------|
| 시나리오1 | Pod A (`10.96.1.102`) | Pod B (`10.96.8.8`) | 8080 | East-West, **동일 노드** |
| 시나리오2 | Pod A (`10.96.1.102`) | Pod X (`10.96.9.9`) | 8080 | East-West, **노드 간(cross-node)** |
| 시나리오3 | Pod A (`10.96.1.102`) | 외부 서버 C (`192.168.56.7`) | 443 | North-South, **외부 egress** |

## 4. tcpdump 관측 지점 (설계요구사항4)

| 시나리오 | 지점 1 | 지점 2 | 지점 3 | 지점 4 |
|----------|--------|--------|--------|--------|
| 시나리오1 | Pod A | worker node 1 | worker node 1 | Pod B |
| 시나리오2 | Pod A | worker node 1 | worker node 3 | Pod X |
| 시나리오3 | Pod A | worker node 1 | 외부 서버 C | - |

> 노드 내부 관측 지점 해석:
> - **Pod 관측(지점1/4):** Pod 네트워크 네임스페이스의 `eth0`(veth) 에서 관측.
> - **worker node 관측:** gateway mode에 따라 실제 관측 인터페이스가 달라짐.
>   - Shared GW(`routingViaHost:false`): `br-ex`(breth0) / 물리 NIC(`ens*`) 기준.
>   - Local GW(`routingViaHost:true`): egress는 관리 포트 `ovn-k8s-mp0` → 호스트 라우팅 → 물리 NIC 순서로 관측.
> - East-West(시나리오1/2)의 노드 간 전송 구간은 **GENEVE(UDP 6081) 캡슐화** 상태로 물리망을 통과합니다. 오버레이 내부(원본 Pod IP)는 노드의 오버레이 종단(genev_sys / ovn-k8s-mp0 내부) 이후에서 관측됩니다.

## 5. 참고용 tcpdump 명령 예시

```bash
# 1) Pod 내부에서 (디버그 컨테이너 or nsenter)
oc debug node/<node> -- chroot /host bash -c 'tcpdump -ni any host 10.96.8.8 and port 8080'

# 2) 노드 물리 NIC (Shared GW egress 관측)
tcpdump -ni ens192 host 192.168.56.7 and port 443

# 3) OVN 관리 포트 (Local GW egress 관측)
tcpdump -ni ovn-k8s-mp0 host 192.168.56.7 and port 443

# 4) GENEVE 오버레이 (노드 간 East-West 캡슐 관측)
tcpdump -ni ens192 udp port 6081
```

> `oc debug node/...` 및 `nsenter`로 Pod netns에 진입하는 방식은 관측 지점에 따라 선택합니다. 실제 인터페이스명(`ens192` 등)은 환경에 따라 다릅니다.
