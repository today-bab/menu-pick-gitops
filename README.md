# menu-pick-gitops

menu-pick을 k8s-lab 클러스터에 배포하는 GitOps 저장소.
Argo CD가 이 저장소를 보고 클러스터를 같은 상태로 맞춘다.

## 구조

```
bootstrap/     처음 한 번만 직접 적용 (Argo CD 설치 + 루트 Application)
apps/          Argo CD Application 목록 (app-of-apps). 루트 Application이 이 폴더를 동기화
platform/      앱이 공통으로 쓰는 클러스터 구성요소
  sealed-secrets/   비밀값 암호화 컨트롤러
  cloudflared/      Cloudflare Tunnel 커넥터 (api.todaybab.com → menu-pick-backend)
  monitoring/       kube-prometheus-stack values, 알림 Secret, Grafana 대시보드
services/      menu-pick 서비스 매니페스트
  menu-pick-backend/   API 서버 (Pod 2개, 수집·경고 규칙 포함)
  menu-pick-valkey/    백엔드 Pod들이 함께 쓰는 캐시 (Redis 호환 Valkey)
```

## 전체 흐름

```
브라우저 ─▶ https://menu.todaybab.com (Vercel)
   └─ API ─▶ https://api.todaybab.com ─▶ Cloudflare Tunnel ─▶ cloudflared ─▶ menu-pick-backend:8080

backend push ─▶ CI(테스트·GHCR 이미지) ─▶ 이 저장소의 이미지 태그 커밋 ─▶ Argo CD 배포
```

## 캐시 (Valkey)

- 백엔드는 카카오에서 받은 근처 음식점 목록을 좌표 칸(약 100m)별로 10분 동안 Valkey에 저장해, 두 Pod가 함께 쓴다.
- 캐시 전용이라 디스크에 저장하지 않는다 (`maxmemory 48mb`, `allkeys-lru`). 재시작하면 비지만 다시 채워진다.
- 비밀번호 대신 NetworkPolicy로 백엔드 Pod만 접속할 수 있다.
- Valkey가 죽어도 백엔드는 캐시 없이 카카오를 불러 정상 응답한다.

```sh
# 저장된 키 보기
kubectl -n menu-pick exec deploy/menu-pick-valkey -- valkey-cli --scan --pattern 'nearby-restaurants*'
```

## 운영 원칙

- 클러스터에 무엇을 배포할지는 `apps/`에 Application을 추가해서 정한다.
- 비밀값은 평문으로 커밋하지 않는다. Sealed Secrets로 암호화한 `SealedSecret`만 커밋한다.

## 클러스터 접근

클러스터 API(`10.10.10.10:6443`)는 홈 서버 안쪽 대역이라 SSH 터널로 접근한다.

```sh
# 집 LAN이면 home1, 밖이면 home2
ssh -N -L 6443:10.10.10.10:6443 home1
```

kubeconfig의 서버를 `https://127.0.0.1:6443`, `tls-server-name: k8s-master`로 둔 context를 쓴다.

## 처음 설치 (클러스터를 새로 만들었을 때)

```sh
kubectl apply --server-side -k bootstrap/argocd
kubectl -n argocd rollout status deploy/argocd-server
kubectl apply -f bootstrap/root-app.yaml
```

이후로는 Argo CD가 `apps/`를 보고 나머지(Argo CD 자신, Sealed Secrets, 앱)를 설치한다.

## Argo CD UI

외부에 공개하지 않고 port-forward로 연다.

```sh
kubectl -n argocd port-forward svc/argocd-server 8443:443
# https://localhost:8443 , 사용자 admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

로그인한 뒤 비밀번호를 바꾸고 `argocd-initial-admin-secret`은 지운다.

## 모니터링

| 무엇 | 어디서 | 알림 |
| ---- | ------ | ---- |
| 바깥에서 본 생존 여부 (API, 웹) | UptimeRobot, 5분 간격 | Discord |
| 클러스터·백엔드 지표와 경고 | Prometheus + Alertmanager (`monitoring`) | Discord |
| 대시보드 | Grafana (`monitoring`) | - |

집 서버가 통째로 꺼지면 클러스터 안의 경고도 함께 멈추므로, 바깥 확인은 UptimeRobot이 맡는다.

### 백엔드 지표

- 백엔드는 관리 포트 8081에서 `/actuator/prometheus`를 연다. 터널은 8080만 연결하므로 지표는 외부에 공개되지 않는다.
- `services/menu-pick-backend/servicemonitor.yaml`이 30초마다 수집한다.
- 경고 규칙(`prometheusrule.yaml`): 백엔드 다운(critical), 5xx 비율 5% 초과, 카카오 호출 실패율 20% 초과, 평균 응답 2초 초과.
  Pod 반복 재시작·노드 자원 부족 같은 일반 경고는 kube-prometheus-stack 기본 규칙이 맡는다.

### Grafana

**https://grafana.todaybab.com** (Cloudflare Tunnel)

- 앞단의 Cloudflare Access(Zero Trust > Access controls > Applications > grafana, 정책 `me only`)가 허용한 이메일만 들어올 수 있다. 이메일로 받은 일회용 코드로 로그인한 뒤 Grafana `admin`으로 다시 로그인한다.
- 터널 경로: `grafana.todaybab.com` → `http://kube-prometheus-stack-grafana.monitoring.svc.cluster.local:80` (Cloudflare 대시보드에서 관리)
- 경로를 새로 열 때는 Access 애플리케이션을 먼저 만들고 터널 경로를 나중에 추가한다. 순서가 바뀌면 그 사이 보호 없이 공개된다.

클러스터 안에서 바로 열 때:

```sh
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
# http://localhost:3000 , 사용자 admin
```

관리자 비밀번호는 `platform/monitoring/resources/grafana-admin.sealed-secret.yaml`에 암호화되어 있다.
Grafana는 디스크에 저장하지 않으므로 화면에서 비밀번호를 바꾸면 재시작할 때 이 값으로 돌아간다.
바꿀 때는 `platform/monitoring/resources/kustomization.yaml`의 주석대로 SealedSecret을 다시 만든다.

- `menu-pick` 폴더: 서비스 대시보드 (`platform/monitoring/resources/dashboards/menu-pick.json`)
- `Spring Boot` 폴더: JVM (Micrometer), Spring Boot 3.x Statistics (grafana.com 공개 대시보드)

대시보드를 고칠 때는 Grafana에서 편집한 뒤 JSON을 내보내 이 저장소의 파일을 바꾼다. Grafana는 디스크에 저장하지 않으므로 화면에서만 고친 내용은 재시작하면 사라진다.

### 알림 (Discord)

Alertmanager는 warning·critical 경고를 Discord로 보내고, 해결되면 해결 알림도 보낸다.
웹훅 URL은 `platform/monitoring/resources/alertmanager-discord.sealed-secret.yaml`에 암호화되어 있다.

시험 경고 보내기:

```sh
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093
curl -H 'Content-Type: application/json' -d '[{"labels":{"alertname":"Test","severity":"warning"}}]' localhost:9093/api/v2/alerts
```
