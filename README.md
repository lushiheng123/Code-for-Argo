# 结构说明
![alt text](README_Images/README/{2665AA99-66BA-423D-83D0-0DAF19B5AB2B}.png)

# 先装Argocd
```sh
# 手动删除 dex 的 deployment
kubectl delete deployment -n argocd argocd-dex-server

# 然后升级配置禁用 dex
helm upgrade argocd argo/argo-cd \
  --namespace argocd \
  --values values.yaml \
  --version 7.7.15 \
  --set dex.enabled=false
```
![alt text](README_Images/README/{3AD59815-F17E-4DA6-9D76-1180F08C617B}.png)

#  部署 monitoring

# 连接GIT分支
```sh
仓库地址
https://github.com/lushiheng123/Code-for-Argo
