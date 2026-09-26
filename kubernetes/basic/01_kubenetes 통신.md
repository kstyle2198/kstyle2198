# Kubernetes와 통신

쿠버네티스와 통신하기 위한 설정 파일 : ~/.kube/config

```
# kube info
kubectl cluster-info

# 현재 config 확인
kubectl config get-contexts
# 또는
kubectl config current-context

# 특정 config 전환
kubectl config use-context docker-desktop

# ~/.kube/config 파일 내 아래와 같이 current-context가 정의되어 있다.
`
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop

- context:
    cluster: k3d-j-cluster
    user: admin@k3d-j-cluster
  name: k3d-j-cluster

current-context: k3d-j-cluster
`
```
