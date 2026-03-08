# GitOps 环境搭建

这是一个按环境拆分应用清单的 GitOps 仓库。

如果你是第一次接手这个仓库，**建议先用 `kind` 跑通一遍完整流程**，确认 Argo CD、ApplicationSet、应用渲染都正常，再去看目录结构和维护方式。

## 先跑通 kind

### 1. 创建本地 kind 集群
```bash
# 创建集群命令
kind create cluster --config ./kind-config.yaml --name huari-test --image kindest/node:v1.34.0 --retain
# 切换kubectl上下文
sudo kubectl cluster-info --context kind-huari-test
# 查看集群节点
sudo kubectl get nodes
# 查看集群全部的pod
sudo kubectl get pods -A -owide
# 删除集群
sudo kind delete cluster --name huari-test
```

### 2. 安装 Argo CD

```shell
# 更新helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 部署argocd
helm upgrade --install argocd argo/argo-cd \
  --version 7.3.5 \
  --namespace argo \
  --create-namespace
  
# 使用端口转发访问 UI
kubectl port-forward svc/argocd-server -n argo 8080:443 --address 0.0.0.0

# 获取初始管理员密码
kubectl -n argo get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 3. 配置 Git 仓库凭据（如果仓库是私有的）

如果代码仓库或 values 仓库是私有仓库，需要先在 Argo CD 配置仓库 Secret。

示例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-gitlab-values
  namespace: argo
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://gitlab.com/<group>/<repo>.git
  username: <gitlab-username-or-token-name>
  password: <gitlab-token-with-read_repository>
```

```bash
kubectl apply -f repo-gitlab-values.yaml
```

### 4. 替换仓库占位符

请至少替换以下文件中的占位符仓库地址：
- `deploy/kind.yaml`
- `deploy/prod.yaml`
- `environments/kind.yaml`
- `environments/prod.yaml`

然后将变更推送到 Git 仓库！！！

### 5. 部署 kind 环境

```bash
helm template gitops-bootstrap chart \
  -f deploy/kind.yaml | kubectl apply -f -
```

### 6. 验证部署结果

先看 Argo CD 是否已经生成应用：

```bash
kubectl get applications -n argo
kubectl get applicationsets -n argo
kubectl get appprojects -n argo
```

再看关键命名空间资源：

```bash
kubectl get pods -n argo
kubectl get pods -n ingress-nginx
kubectl get pods -n harbor
```

说明：
- `kind` 默认关闭了 `gitlab`，所以本地验证时看不到 `gitlab` 相关资源是正常的
- 默认情况下会先验证整套 GitOps 链路是否打通，而不是追求一次性把所有重型组件都跑起来

## 再部署 prod 示例

当 `kind` 跑通以后，再看 `prod` 示例会更容易理解：

```bash
helm template gitops-bootstrap chart \
  -f deploy/prod.yaml | kubectl apply -f -
```

## 常用命令

### 创建本地 kind 集群

```bash
kind create cluster --config ./kind-config.yaml
```

### 部署 kind

```bash
helm template gitops-bootstrap chart \
  -f deploy/kind.yaml | kubectl apply -f -
```

### 部署 prod 示例

```bash
helm template gitops-bootstrap chart \
  -f deploy/prod.yaml | kubectl apply -f -
```

### 查看 Argo CD 应用

```bash
kubectl get applications -n argo
```

## 仓库结构

```text
.
├── chart/
├── deploy/
│   ├── kind.yaml
│   └── prod.yaml
├── apps/
│   ├── kind/
│   └── prod/
├── environments/
│   ├── kind.yaml
│   └── prod.yaml
├── values/
│   ├── kind/
│   └── prod/
├── kind-config.yaml
└── README.md
```

## 这 5 个目录分别管什么

### `deploy/`

`deploy/` 是正式部署入口。

这里存放的是 **bootstrap chart 的环境级部署配置**，例如：
- `project.name`
- `appSet.name`
- `environment.file`
- `appSet.appsFile`
- `generatorRepo`

也就是说，真正执行部署时，优先使用：
- `deploy/kind.yaml`
- `deploy/prod.yaml`

而不是直接把正式部署配置放进 `chart/` 目录里。

### `apps/<env>/`

控制这个环境下：
- 部署哪些应用
- `enabled`
- `version`
- `syncWave`
- `destinationNamespace`
- chart 来源

示例：`apps/kind/gitlab.yaml`

```yaml
name: gitlab
enabled: "false"
repoURL: https://charts.gitlab.io
chart: gitlab
version: 8.8.1
destinationNamespace: gitlab
syncWave: "4"
```

### `values/<env>/`

控制应用自身参数，也就是子 chart 的 values。

示例：`values/kind/harbor-values.yaml`

```yaml
expose:
  type: ingress
  tls:
    enabled: false
```

### `environments/<env>.yaml`

控制环境公共信息。

示例：`environments/kind.yaml`

```yaml
env: kind
valuesRepoURL: https://gitlab.papegames.com/huari/gitops-environment-setup.git
valuesRevision: main
valuesDir: kind
clusterServer: https://kubernetes.default.svc
```

### `chart/`

`chart/` 只保留 Helm Chart 本身：
- `Chart.yaml`
- `values.yaml`
- `templates/`

这里的 `chart/values.yaml` 是 **chart 默认值**，不是正式环境部署入口。

## 渲染逻辑

### 第一步：渲染启动层

执行：

```bash
helm template gitops-bootstrap chart -f deploy/kind.yaml
```

会生成：
- `AppProject`
- `ApplicationSet`

### 第二步：ApplicationSet 生成应用

`ApplicationSet` 会：
1. 读取 `environment.file`
2. 读取 `appSet.appsFile`
3. 过滤 `enabled == true` 的应用
4. 生成最终的 `Application`
5. 加载 `values/<env>/<app>-values.yaml`

## 当前默认约定

- `deploy/kind.yaml` 指向：
  - `environment.file=environments/kind.yaml`
  - `appSet.appsFile=apps/kind/*.yaml`
- `deploy/prod.yaml` 指向：
  - `environment.file=environments/prod.yaml`
  - `appSet.appsFile=apps/prod/*.yaml`

### kind 与 prod 的默认差异

| 项目 | kind | prod |
| --- | --- | --- |
| 部署入口 | `deploy/kind.yaml` | `deploy/prod.yaml` |
| 环境文件 | `environments/kind.yaml` | `environments/prod.yaml` |
| 应用目录 | `apps/kind/` | `apps/prod/` |
| Harbor values 文件 | `values/kind/harbor-values.yaml` | `values/prod/harbor-values.yaml` |
| GitLab 默认状态 | `false` | `true` |

## 日常维护

### 新增应用

通常要新增 4 个文件：
1. `apps/kind/<app>.yaml`
2. `apps/prod/<app>.yaml`
3. `values/kind/<app>-values.yaml`
4. `values/prod/<app>-values.yaml`

### 修改版本

直接改对应环境的 app 文件：
- `apps/kind/harbor.yaml`
- `apps/prod/harbor.yaml`

### 开关应用

直接改对应环境 app 文件里的 `enabled`。

### 调整应用参数

直接改对应环境的 values 文件：
- `values/kind/<app>-values.yaml`
- `values/prod/<app>-values.yaml`

### 新增环境

新增环境时补齐这 4 类文件：
1. `deploy/<env>.yaml`
2. `environments/<env>.yaml`
3. `apps/<env>/*.yaml`
4. `values/<env>/*-values.yaml`

## 常见排查命令

### 查看 Argo CD 资源

```bash
kubectl get applications -n argo
kubectl get appprojects -n argo
kubectl get applicationsets -n argo
```

### 查看 Argo CD 组件状态

```bash
kubectl get pods -n argo
kubectl get svc -n argo
```

### 查看目标命名空间资源

```bash
kubectl get pods -n ingress-nginx
kubectl get pods -n harbor
kubectl get pods -n gitlab
```

### 查看应用详情与事件

```bash
kubectl describe application kind-harbor -n argo
kubectl describe application prod-harbor -n argo
```

### 查看控制器日志

```bash
kubectl logs -n argo deploy/argocd-application-controller
kubectl logs -n argo deploy/argocd-server
```

## 多集群说明

- 目标集群由 `environments/<env>.yaml` 的 `clusterServer` 决定
- `https://kubernetes.default.svc` 表示 Argo CD 所在集群
- 如果要发到其它集群：
  1. 在 Argo CD 注册目标集群
  2. 修改 `environments/<env>.yaml` 的 `clusterServer`
  3. 确保 `project.destinations` 放行该集群
