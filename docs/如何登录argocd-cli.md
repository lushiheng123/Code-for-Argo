# 登录
```sh
argocd login localhost:8080 \
--username admin \
--password admin123 \
--insecure
```

哈哈，终于登录成功了🎉。

其实你这次遇到的问题非常典型，也是很多刚学 Argo CD 的人都会踩的坑。下面我按照**企业排障思路**给你总结，以后遇到 CLI 登录问题基本都可以按这个流程定位。

---

# ArgoCD CLI 登录排障思路

## 第一步：确认 ArgoCD 本身是否正常

先不要急着登录 CLI，确认服务是否正常运行。

### 查看 Pod

```bash
kubectl get pods -n argocd
```

应该全部是

```text
Running
```

例如

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-redis
```

如果这里都有问题，CLI 肯定登录不了。

---

## 第二步：确认 Server 是否能访问

查看 Service

```bash
kubectl get svc -n argocd
```

例如

```text
argocd-server
NodePort
80 -> 30080
443 -> 30443
```

如果你访问的是

```text
localhost:8080
```

说明应该有

```bash
kubectl port-forward svc/argocd-server \
-n argocd \
8080:80
```

如果访问的是

```text
localhost:30080
```

说明直接访问 NodePort。

这一点必须先弄清楚。

---

## 第三步：确认 Server 配置

查看 Deployment

```bash
kubectl get deployment argocd-server \
-n argocd -o yaml
```

重点看有没有

```text
--insecure
```

如果没有：

CLI 默认会走 HTTPS。

如果有：

CLI 默认就是 HTTP。

你的就是：

```text
--insecure
```

所以：

```bash
argocd login localhost:8080
```

才会提示：

```text
server is not configured with TLS
```

这是正常现象。

---

# 第四步：确认密码来源

这里是你这次最大的坑。

很多教程都是：

```bash
kubectl get secret \
argocd-initial-admin-secret
```

但是：

**只有没有指定密码的时候，ArgoCD 才会生成这个 Secret。**

而你安装的时候：

```yaml
configs:
  secret:
    argocdServerAdminPassword: "$2a$10...."
```

所以：

```text
argocd-initial-admin-secret
```

根本不存在。

因此：

```bash
kubectl get secret argocd-initial-admin-secret
```

一直报：

```text
NotFound
```

这是正常行为。

---

# 第五步：确认 bcrypt 是否正确

ArgoCD 保存的是

```text
bcrypt hash
```

例如

```text
$2a$10....
```

它不知道你的原始密码。

如果：

```yaml
argocdServerAdminPassword:
"$2a$10...."
```

对应的是：

```text
admin123
```

那么登录必须输入：

```text
admin123
```

而不是：

```text
$2a$10....
```

也不是：

```text
admin
```

---

推荐生成方式：

```bash
argocd account bcrypt \
--password admin123
```

得到：

```text
$2a$10....
```

然后放进 Helm values。

---

# 第六步：更新 Helm

修改 values 后：

```bash
helm upgrade argocd argo/argo-cd \
-n argocd \
-f values.yaml
```

不要：

```text
kubectl edit secret
```

因为：

Helm 管理的资源最好还是让 Helm 管理。

---

## 第七步：等待 Deployment 更新

查看：

```bash
kubectl rollout status deployment/argocd-server \
-n argocd
```

确认已经滚动完成。

---

# 第八步：登录 CLI

HTTP 模式：

```bash
argocd login localhost:8080
```

如果需要：

```bash
argocd login localhost:8080 \
--username admin \
--password admin123 \
--insecure
```

有些环境（例如 Ingress、反向代理）需要：

```bash
--grpc-web
```

或者

```bash
--plaintext
```

但你这里最终是不需要的。

---

# 以后遇到 CLI 登录问题，我建议按下面这个顺序排查

```text
① Pod 是否 Running？
        │
        ▼
② Service 是否能访问？
        │
        ▼
③ argocd-server 是否启动成功？
        │
        ▼
④ 是否开启 --insecure？
        │
        ▼
⑤ 当前访问的是 HTTP 还是 HTTPS？
        │
        ▼
⑥ admin 密码来自哪里？
        │
        ├── initial-admin-secret
        │
        └── Helm values（bcrypt）
        │
        ▼
⑦ 修改 values 后是否执行 helm upgrade？
        │
        ▼
⑧ Deployment 是否滚动更新？
        │
        ▼
⑨ argocd login
```

---

## 你这次真正的问题归纳

实际上，你的 **Argo CD 服务从一开始就是正常的**：

* ✅ Pod 正常运行。
* ✅ NodePort 配置正确。
* ✅ `--insecure` 配置正确。
* ✅ CLI 能够连接到服务。

真正的问题集中在**认证**：

1. 你在 `values.yaml` 中配置了 `configs.secret.argocdServerAdminPassword`，因此不会生成 `argocd-initial-admin-secret`，继续按网上很多教程去获取这个 Secret 就会失败。
2. 修改了 `values.yaml` 后，需要执行 `helm upgrade` 才会把新的 bcrypt 密码应用到集群。
3. 登录时必须使用 **bcrypt 对应的原始密码**，而不是 bcrypt 字符串本身。

---

我建议你把这次经历当作一次完整的 **Helm 管理应用配置 + Kubernetes 排障** 练习。今后学习像 Prometheus、Grafana、GitLab、Harbor 等 Helm Chart 时，也可以采用同样的排障思路：**先确认工作负载正常，再确认 Service，再确认配置是否生效，最后再检查业务层（这里就是认证）**。这种思路比记命令更重要，也更符合实际生产环境中的排障流程。
