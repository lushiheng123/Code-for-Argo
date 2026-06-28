# ArgoCD & Prometheus Learning - App of Apps Pattern

这是一个用 **App of Apps** 模式组织的 ArgoCD 学习项目。

## 目录结构

```
├── apps/                          # 第一阶段：所有 Helm Charts
│   ├── monitoring/                # Prometheus & Grafana Helm Chart
│   │   ├── Chart.yaml
│   │   ├── values/
│   │   └── templates/
│   └── nginx-exporter/            # NGINX Prometheus Exporter Chart
│       ├── Chart.yaml
│       ├── values/
│       └── templates/
│
├── argocd/                        # 第二阶段：ArgoCD Application 定义
│   ├── monitoring.yaml            # Application: monitoring
│   └── nginx-exporter.yaml        # Application: nginx-exporter
│
├── root-app.yaml                  # 第三阶段：Root Application (App of Apps)
│
├── docs/                          # 学习文档和笔记
│   ├── 启动.md
│   ├── 1-更新argocd.md
│   ├── 如何登录argocd-cli.md
│   └── 添加nginx应用.md
│
└── README.md                      # 本文件
```

## 部署方式

### 方式一：直接部署单个应用（第一、二阶段）

```bash
# 部署 monitoring
kubectl apply -f argocd/monitoring.yaml

# 部署 nginx-exporter
kubectl apply -f argocd/nginx-exporter.yaml
```

### 方式二：App of Apps - 一条命令部署所有（第三阶段）

```bash
# 只需要一条命令，ArgoCD 会自动读取 argocd/ 目录下的所有 Application
kubectl apply -f root-app.yaml
```

当执行这条命令后，ArgoCD 会：
1. 读取 `argocd/` 目录下的所有 `*.yaml` 文件
2. 自动创建 `monitoring` 和 `nginx-exporter` Application
3. 触发 Helm Chart 部署到对应的命名空间

## 学习阶段

| 阶段 | 目标 | 命令 |
|------|------|------|
| 第一阶段 | 学习 Helm Chart 结构 | `helm dependency update && helm install` |
| 第二阶段 | 学习 ArgoCD Application | `kubectl apply -f argocd/*.yaml` |
| 第三阶段 | 学习 App of Apps 模式 | `kubectl apply -f root-app.yaml` |

## 关键要点

### 🎯 App of Apps 的优势

- ✅ **集中管理**：所有应用在一个 Root Application 下
- ✅ **自动化**：一条命令部署所有应用
- ✅ **可扩展**：添加新应用只需在 `argocd/` 下创建新的 YAML
- ✅ **企业级**：这是大型 Kubernetes 集群的标准做法

### 📝 Application 配置说明

每个 Application 都遵循这个模板：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <应用名>           # ArgoCD 中的应用名
  namespace: argocd
spec:
  source:
    repoURL: <Git仓库URL>
    path: apps/<应用名>    # Helm Chart 在仓库中的路径
    targetRevision: <分支名>
  destination:
    server: https://kubernetes.default.svc
    namespace: <部署命名空间>
  syncPolicy:
    automated:
      prune: true          # 自动删除不在 Git 中的资源
      selfHeal: true       # 自动修复漂移的资源
    syncOptions:
      - CreateNamespace=true  # 自动创建命名空间
```

## 后续扩展

添加新应用的步骤：

1. 在 `apps/` 下创建新的 Helm Chart 目录
2. 在 `argocd/` 下创建对应的 `<应用名>.yaml` Application 定义
3. Push 到 Git 仓库，ArgoCD 会自动检测并同步

```bash
# 例如：添加 Redis 应用
1. mkdir -p apps/redis && cp -r apps/nginx-exporter/* apps/redis/
2. cat > argocd/redis.yaml << EOF
   # 复制 nginx-exporter.yaml 并修改名称和路径
   EOF
3. git add . && git commit -m "feat: add redis application"
4. git push
```

## 常用命令

```bash
# 查看所有 Application 状态
kubectl get applications -n argocd

# 查看特定应用的详细信息
kubectl describe app monitoring -n argocd

# 手动同步应用
argocd app sync monitoring

# 查看应用日志
kubectl logs -n argocd deployment/argocd-application-controller

# 获取 ArgoCD 管理员密码
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 参考资源

- [ArgoCD 官方文档](https://argo-cd.readthedocs.io/)
- [ArgoCD App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/app-of-apps/)
- [Helm 官方文档](https://helm.sh/)

---

**学习建议**：按照三个阶段逐步学习，理解每一步的核心概念，最终掌握企业级的 GitOps 部署模式。
