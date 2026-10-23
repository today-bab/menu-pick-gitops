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
