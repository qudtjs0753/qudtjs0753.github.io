---
title: "Kubernetes 통신은 어떤 컴포넌트들이 어떻게 상호작용할까"
date: 2026-04-29 00:00:00 +0900
categories: [Kubernetes]
tags: [CoreDNS, kube-proxy, Service, Networking]
---

> 이 글은 AI가 작성하고 사람이 검토했습니다.

## 들어가며

이 글에서는 **Service가 실제로 무엇인지**, 그리고 **각 컴포넌트가 Service를 어떻게 활용하는지**를 정리한다.
설정 덩어리라고 하는데 처음엔 Pod랑 비슷하게 프로세스가 띄워져있는줄 알았다 ㅎ..
나중엔 CNI에 대해서도 다뤄볼 예정.


---

## Service는 프로세스가 아니다

결론부터 말하면, Service 오브젝트는 **etcd에 저장된 선언 덩어리**다.
직접 트래픽을 받거나 처리하는 프로세스가 아니다.

Service가 생성되면 그 정보를 읽어가는 컴포넌트들이 각자 역할을 수행한다.

```
kubectl apply -f web-svc.yaml
    │
    └── etcd에 Service 오브젝트 저장
          │
          ├── CoreDNS          → DNS 이름 등록
          ├── Endpoints Controller → Pod IP 수집
          └── kube-proxy       → iptables 규칙 생성
```

---

## 각 컴포넌트의 역할

| 컴포넌트 | Service에서 읽는 필드 | 하는 일 |
|---------|-------------------|------|
| CoreDNS | `name`, `namespace`, `clusterIP` | `web-svc.default.svc.cluster.local → Cluster IP` 등록 |
| Endpoints Controller | `selector` | selector에 맞는 Pod IP 수집 → Endpoints 오브젝트 생성/갱신 |
| kube-proxy | `clusterIP`, `ports`, `Endpoints` | iptables 규칙 생성 — Cluster IP로 온 패킷을 Pod IP로 DNAT |

### CoreDNS

클러스터 내부 전용 DNS 서버로, 클러스터에 보통 2개의 replica로 떠 있다.
Pod가 생성될 때 `/etc/resolv.conf`에 CoreDNS 주소가 자동으로 주입되기 때문에, Pod 안에서 DNS 조회를 하면 무조건 CoreDNS를 먼저 거친다.

```
# Pod 안 /etc/resolv.conf
nameserver 10.96.0.10  ← CoreDNS 주소
```

- 클러스터 내부 이름(`web-svc`)이면 직접 Cluster IP를 반환
- 외부 도메인(`google.com`)이면 upstream DNS(8.8.8.8 등)로 중계

### Endpoints Controller

`kube-controller-manager` 안에 있는 컨트롤러다.
Service의 `selector`를 보고 조건에 맞는 Pod IP를 계속 감시하며, 변화가 생기면 Endpoints 오브젝트를 갱신한다.

### kube-proxy

**노드마다 1개씩** DaemonSet으로 배포된다.
Endpoints 오브젝트를 감시하다가 변경이 생기면 iptables(또는 ipvs) 규칙을 갱신해서 트래픽을 실제 Pod IP로 전달한다.

이름이 "proxy"지만, 실제로는 **프록시 서버가 아니다.** 자세한 내용은 아래 섹션에서 다룬다.

---

## Pod가 죽었다 살아났을 때

Pod가 재시작되면 IP가 새로 할당된다. 이때 매핑이 갱신되는 흐름은 아래와 같다.

```
Pod 재시작 → 새 IP 할당
    │
    └── Endpoints Controller가 감지
          │
          └── Endpoints 오브젝트 업데이트 (새 Pod IP로 교체)
                │
                └── kube-proxy가 변경된 Endpoints를 읽어
                      iptables 규칙 갱신
```

CoreDNS는 Cluster IP만 반환하면 되기 때문에 Pod IP가 바뀌어도 신경 쓰지 않는다.
Pod IP 추적은 전부 **Endpoints Controller → kube-proxy** 라인에서 처리된다.

---

## 외부 요청이 들어올 때 / 나갈 때

### 외부 → 클러스터 (들어오는 요청)

```
외부 클라이언트
    │
    └── 외부 DNS → NodePort 또는 LoadBalancer IP
          │
          └── kube-proxy → iptables → Pod IP
```

이 경우 CoreDNS는 관여하지 않는다. 이미 IP로 들어오기 때문이다.

### 클러스터 → 외부 (나가는 요청)

```
Pod
    │
    └── google.com 조회
          │
          └── CoreDNS (resolv.conf에 등록된 유일한 DNS 서버)
                │
                └── upstream DNS → IP 반환 → Pod
```

Pod의 `/etc/resolv.conf`에는 CoreDNS 주소만 등록되어 있어서, 외부 도메인 조회도 CoreDNS를 거쳐 나간다.

---

## Service 타입별 접근 범위

| 타입 | 접근 범위 | 주요 용도 |
|------|---------|---------|
| ClusterIP | 클러스터 내부만 | 서비스끼리 내부 통신 (DB, 캐시 등) |
| NodePort | 노드 IP + 포트로 외부 접근 | 개발/테스트 |
| LoadBalancer | 외부 로드밸런서 IP | 프로덕션 외부 서비스 |

> **Kind 환경 주의**: NodePort 접근 시 `127.0.0.1`이 아닌 `kubectl get nodes -o wide`의 `INTERNAL-IP`를 사용해야 한다. Kind 노드는 Docker 컨테이너로 실행되기 때문이다.

---

## Pod끼리 통신할 때

Pod 간 통신에는 두 가지 경로가 있다.

### 경로 1: Service 이름으로 통신 (권장)

```
Pod A (app=frontend)
    │
    └── "web-svc"로 요청
          │
          ▼
       CoreDNS
          │  web-svc.default.svc.cluster.local → 10.96.x.x (Cluster IP) 반환
          ▼
       Pod A의 커널 (iptables)
          │  kube-proxy가 심어둔 DNAT 규칙 적용
          │  10.96.x.x → 10.244.x.x (Pod B IP)
          ▼
       CNI
          │  목적지 Pod IP까지 실제 패킷 라우팅
          ▼
       Pod B (app=web)
```

각 컴포넌트가 맡는 역할:

| 컴포넌트 | 역할 |
|---------|------|
| CoreDNS | Service 이름 → Cluster IP 변환 |
| kube-proxy | Cluster IP → Pod IP DNAT 규칙 유지 |
| Endpoints Controller | kube-proxy가 쓸 Pod IP 목록을 최신 상태로 갱신 |
| CNI | 최종 패킷을 목적지 Pod까지 라우팅 |

### 경로 2: Pod IP 직접 통신

```
Pod A
    │
    └── 10.244.1.5 (Pod B IP) 직접 요청
          │
          ▼ CNI 플러그인이 Pod 간 라우팅 처리
       Pod B
```

CoreDNS와 kube-proxy는 관여하지 않는다. CNI만 경로를 처리한다.

단, Pod IP는 재시작 시 바뀌므로 하드코딩이 불가능하다. 실제로는 거의 쓰지 않는 방식이다.

### 두 경로 비교

| | Service 이름 | Pod IP 직접 |
|--|------------|------------|
| CoreDNS | 이름→IP 변환 | 불필요 |
| kube-proxy | Cluster IP→Pod IP DNAT | 불필요 |
| Endpoints Controller | Pod IP 목록 갱신 | 불필요 |
| CNI | 최종 라우팅 | 라우팅 |
| Pod 재시작 후 동작 | 정상 (IP 자동 갱신) | 중단 (IP 바뀜) |

---

## kube-proxy는 진짜 proxy가 아니다

이름 때문에 혼동하기 쉽지만, kube-proxy는 **트래픽을 직접 받아서 전달하는 프록시 서버가 아니다.**

kube-proxy 프로세스가 하는 일은 딱 하나다: **리눅스 커널의 iptables 규칙을 써놓는 것.**

```
kube-proxy 프로세스
    │
    └── Endpoints 변경 감지
          │
          └── iptables 규칙 업데이트 (한 번 쓰고 끝)

이후 트래픽은 커널이 직접 처리:
Pod A → Cluster IP 패킷
    │
    └── 커널 netfilter (iptables)
          │  DNAT 규칙: 10.96.x.x → 10.244.x.x
          ▼
       Pod B (직접 도달)
```

규칙이 한 번 심어지면, 이후 패킷 경로에 kube-proxy 프로세스는 **없다.** 커널이 직접 DNAT을 수행한다.

| | 트래픽 직접 처리 | 역할 |
|--|--------------|------|
| nginx 같은 전통 proxy | O | 패킷을 받아서 전달 |
| kube-proxy | X | iptables 규칙을 관리하는 설정기 |
| CoreDNS | O (DNS만) | DNS 쿼리를 직접 받아서 응답 |

kube-proxy라는 이름이 붙은 이유는 초기 구현이 실제 userspace proxy였기 때문이다. 성능 문제로 iptables/ipvs 방식으로 전환됐는데, 이름만 그대로 남았다.

---

## 정리

Service는 세 컴포넌트가 각자 필요한 정보를 가져가는 **단일 선언 지점**이다.

- **CoreDNS**: 이름 → Cluster IP 변환
- **Endpoints Controller**: selector로 Pod IP 수집 및 갱신
- **kube-proxy**: Cluster IP → Pod IP 트래픽 전달

Service 자체가 트래픽을 처리하는 게 아니라, 이 세 컴포넌트가 협력해서 "이름으로 안정적으로 접근 가능한 엔드포인트"를 만들어내는 구조다.

---

## 참고 자료

- [Kubernetes 공식 문서 — Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes 공식 문서 — DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes 공식 문서 — kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
