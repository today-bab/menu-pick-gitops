# menu-pick-gitops

menu-pick을 k8s-lab 클러스터에 배포하는 GitOps 저장소.
Argo CD가 이 저장소를 보고 클러스터를 같은 상태로 맞춘다.

## 구조

```
bootstrap/     처음 한 번만 직접 적용 (Argo CD 설치 + 루트 Application)
apps/          Argo CD Application 목록 (app-of-apps). 루트 Application이 이 폴더를 동기화
platform/      앱이 공통으로 쓰는 클러스터 구성요소 (Sealed Secrets 등)
```

- 클러스터에 무엇을 배포할지는 `apps/`에 Application을 추가해서 정한다.
- 비밀값은 평문으로 커밋하지 않는다. Sealed Secrets로 암호화한 `SealedSecret`만 커밋한다.

## 클러스터 접근

클러스터 API(`10.10.10.10:6443`)는 홈 서버 안쪽 대역이라 SSH 터널로 접근한다.

```sh
# 집 LAN이면 home1, 밖이면 home2
ssh -N -L 6443:10.10.10.10:6443 home1
```

kubeconfig의 서버를 `https://127.0.0.1:6443`, `tls-server-name: k8s-master`로 둔 context를 쓴다.
