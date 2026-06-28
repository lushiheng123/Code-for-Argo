# 添加repo 
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
![alt text](README_Images/README/{01110210-09F5-4B2D-93E7-7498BD7D6B35}.png)
# 查看所有版本
helm search repo prometheus-community

# 只看最新几个
helm search repo prometheus-community|grep -i nginx
```sh
prometheus-community/prometheus-nginx-exporter          1.22.5          1.5.1           A Helm chart for NGINX Prometheus Exporter
```
# 方法一：不带版本号，拉最新版
helm pull prometheus-community/prometheus-nginx-exporter --untar

# 方法二：指定版本
helm pull bitnami/redis --version 27.0.12 --untar

# 渲染输出
# 用默认 values 渲染
helm template my-redis . > default-output.yaml

# 或者用你自己的 values 覆盖
helm template my-redis . -f my-values.yaml > my-output.yaml

# 渲染到目录（每个资源一个文件）
helm template my-nginx prometheus-community/prometheus-nginx-exporter \
  --namespace myapp \
  --output-dir ./rendered
```sh
helm dependency update
```
```sh
argocd app create nginx-exporter \
  --repo https://github.com/lushiheng123/Code-for-Argo.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace nginx-exporter \
  --revision nginx-prometheus-exporter \
  --sync-policy automated
```