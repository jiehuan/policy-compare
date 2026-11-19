# 通过 Harness 部署到 OpenShift

这条路径与公司其他应用一致：**Harness CI 构建镜像 → 推到 Artifactory → Harness CD（Kubernetes 部署）发布到 OpenShift**。它取代原 `deploy/kubernetes/policy-compare.yaml`。[docs/deployment.md](../../docs/deployment.md) 中的手工 `oc` 流程（多版本路径、固定入口切换、Playwright 验收）仍可用于正式验收，两者不要混用于同一项目。

应用本身只是一个 Node 静态文件服务器（端口 8080，`/healthz` 健康检查）。PDF 在用户浏览器里处理，没有后端接口、数据库或密钥。

## 目录

| 文件 | 用途 |
|---|---|
| `values.yaml` | Harness 解析 `<+artifacts.primary.image>`，以及副本、域名、拉取凭据等可覆盖参数 |
| `templates/policy-compare.yaml` | Deployment + Service + Route（edge TLS），可选 NetworkPolicy |
| `harness-service.yaml` | 粘贴到 Harness 的 Service YAML（不被部署本身读取） |
| `harness-pipeline.yaml` | 粘贴到 Harness 的 Pipeline YAML（不被部署本身读取） |

## 在公司电脑上配置 Harness（预构建方式）

镜像在 Harness 里构建，代码由 Harness 自己从 GitHub 拉取，公司电脑上通常只需要浏览器。下文的分支名 `deploy/harness-openshift` 是当前待验证的分支；合并到 `main` 后改用 `main`（见第 7 步）。

### 1. 确认 Harness 能读到本仓库

- 本仓库之前的 Node 24 冒烟测试（提交 `9696905`）就是在 Harness 里从本仓库构建的。如果那条 pipeline 还在，它用的 GitHub 连接器就能读本仓库，直接沿用，跳到第 2 步。
- 如果 Harness 读不到 `github.com/wlvh/policy-compare`，在公司电脑上用 git 同步到 Harness 能读的仓库：

  ```sh
  git clone --branch deploy/harness-openshift https://github.com/wlvh/policy-compare.git
  cd policy-compare
  git remote add company <公司仓库地址>
  git push company deploy/harness-openshift
  ```

  然后把两份 YAML 里的 `repoName` 改成公司仓库的 `<org>/<repo>`。必须用 git 推送：不要下载 ZIP 再经网页上传，也不要用其他方式复制文件。`prebuilt/` 的文件逐字节校验，换行符变化会让容器启动失败。

### 2. 从 loss-run 抄配置值

在 Harness 中打开 loss-run（golden-loss-tamer）的 pipeline 和 service，切到 YAML 视图，对照下表找到两份 YAML 中 `<...>` 占位符的值：

| 占位符 | loss-run 中的位置 |
|---|---|
| `<ORG_ID>`、`<PROJECT_ID>` | pipeline 顶部的 `orgIdentifier`、`projectIdentifier` |
| `<GITHUB_CONNECTOR_ID>` | 能读本仓库的 GitHub 连接器（见第 1 步）。loss-run 的 `properties.ci.codebase.connectorRef` 只有在它是账号级（Account）连接器时才能复用；若是仓库级连接器，删掉两份 YAML 中的 `repoName` 行 |
| `<BUILD_CLUSTER_CONNECTOR_ID>`、`<BUILD_NAMESPACE>` | CI stage 的 `infrastructure`。建议整块照抄 loss-run 的 `infrastructure` |
| `<ARTIFACTORY_DOCKER_CONNECTOR_ID>` | CI 构建推送步骤的 `connectorRef` |
| `<ARTIFACTORY_DOCKER_HOST>/<DOCKER_LOCAL_REPO>` | 该步骤的 `repo`，去掉最后的镜像名 |
| `<ARTIFACTORY_CONNECTOR_ID>`、`<DOCKER_LOCAL_REPO>`、`<ARTIFACTORY_DOCKER_HOST>`（service 中） | loss-run service 的 primary artifact 来源：`connectorRef`、`repository`、`repositoryUrl` |
| `<ENVIRONMENT_ID>`、`<INFRA_DEFINITION_ID>` | deploy stage 的 `environment.environmentRef` 和 `infrastructureDefinitions[].identifier` |

如果 loss-run 的构建步骤类型或写法不同，以 loss-run 为准，只改 Dockerfile、Context、Tags、Build Arguments 四项（值见第 4 步）。

### 3. 创建 Service

在 GitHub 打开 `deploy/harness/harness-service.yaml`，点 **Raw** 后全选复制。Harness 项目中进入 Services → New Service，切到 YAML 视图，粘贴并替换占位符后保存。

如果项目里已有引用 `deploy/kubernetes/policy-compare.yaml` 的 policy-compare service：该文件在本分支已删除。把它的 manifest 改为类型 K8sManifest、Files `deploy/harness/templates`、Values `deploy/harness/values.yaml`、分支 `deploy/harness-openshift`，或者删掉后按上面新建。`identifier: policycompare` 已被占用时，改成其他值，并同步修改 pipeline 中的 `serviceRef`。

### 4. 创建 Pipeline

复制 `deploy/harness/harness-pipeline.yaml` 的 Raw 内容。Pipelines → Create a Pipeline → 起名 → Inline，切到 YAML 视图，粘贴并替换占位符后保存。

如果沿用已有的、构建本仓库的 pipeline，只需改构建步骤：

| 字段 | 值 |
|---|---|
| Dockerfile | `deploy/Dockerfile.prebuilt` |
| Context | `.` |
| Tags | `r<+pipeline.sequenceId>-<+codebase.shortCommitSha>` |
| Build Arguments | 只保留 `ENGINE: mupdf`，删除其他参数 |

另外，把它的 deploy stage 指向第 3 步的 service。

### 5. 运行

点 Run，Build Type 选 Git Branch，分支填 `deploy/harness-openshift`。

### 6. 检查

- Build stage：日志中镜像以 `r<序号>-<短提交号>` 标签推送到 Artifactory。
- Deploy stage：Rolling Deployment 成功，Pod 就绪。
- 访问地址：在 OpenShift 控制台 Networking → Routes → `policy-compare` 查看 Location，或运行 `oc -n <namespace> get route policy-compare`。
- 浏览器打开该地址，应跳转到 `/releases/<releaseId>/`。预构建方式下，`<releaseId>` 等于 `prebuilt/mupdf/build-info.json` 中的 `releaseId`，与 Harness 镜像标签无关。
- 点页面上的 **P02 Amount change** 示例，结果卡片应显示 `USD 100` 改为 `USD 2000`。

### 7. 合并到 main

跑通后，在 GitHub 上把 `deploy/harness-openshift` 合并到 `main`。然后把 service 中 manifest 的 `branch` 改成 `main`，以后运行时选 `main`。

### 常见问题

| 现象 | 处理 |
|---|---|
| Deploy 阶段找不到 `deploy/harness/templates` | service 的 manifest `branch` 与构建分支不一致，或连接器读的不是本仓库 |
| 拉不到 Node 基础镜像 | 构建参数 `NODE_IMAGE` 改为 loss-run / gdi-poc 所用前缀下可用的 Node 24 镜像 |
| `ImagePullBackOff` | `values.yaml` 填 `imagePullSecret`，用 loss-run 所在 namespace 现有的拉取凭据 |
| 容器启动即退出，日志 `Resource integrity failure` | `prebuilt/` 文件被改过（多半是换行符）。确认 `.gitattributes` 生效，在构建机上重新预构建、复制并提交 |
| Pod 就绪但网址返回 503 | `values.yaml` 设 `networkPolicy.enabled: true` |

## 两种构建方式

| 方式 | Dockerfile | 适用情况 |
|---|---|---|
| **预构建（推荐用于 AXA XL Harness）** | `deploy/Dockerfile.prebuilt` | 构建 Pod 不能上外网。网页在有网络的机器上先构建好，提交到 `prebuilt/mupdf/`；镜像只需要 Artifactory 里的 Node 基础镜像。 |
| 源码构建 | `deploy/Dockerfile` | 构建 Pod 能访问 npm 和 static.rust-lang.org（或其 Artifactory 镜像）。 |

### 预构建步骤

在能上网的 macOS/Linux/WSL 机器上（Node 24、pnpm 10.12.1、Rust 1.93.0 + `wasm32-unknown-unknown`，见根目录 README）：

```sh
pnpm install --frozen-lockfile
pnpm install:mupdf
export POLICY_RELEASE_ID="r$(date -u +%Y%m%d%H%M)-$(git rev-parse --short=7 HEAD)"
POLICY_REVISION="$(git rev-parse HEAD)" POLICY_CONTACT="<维护渠道>" pnpm build:mupdf
rm -rf prebuilt/mupdf && mkdir -p prebuilt && cp -R dist/mupdf prebuilt/mupdf
git add prebuilt/mupdf && git commit -m "Prebuilt mupdf $POLICY_RELEASE_ID"
```

`.gitattributes` 已将 `prebuilt/**` 设为 `-text`：服务器启动时按 `build-info.json` 校验每个文件的 SHA-256，换行符被改动会导致容器启动失败。CI 阶段 Dockerfile 填 `deploy/Dockerfile.prebuilt`，不需要 build arguments（基础镜像默认 `artifactory.platform.axaxl-cloud.com/docker/library/node:24`，已在 Harness/OpenShift Buildah 中验证可拉取；可用 `NODE_IMAGE` 覆盖）。镜像 Tag 建议与 `POLICY_RELEASE_ID` 相同。

## CI 阶段：Build and Push to Docker Registry

预构建方式只需填 Dockerfile `deploy/Dockerfile.prebuilt`、Context、Repo 和 Tag，下面的 Build arguments 表不适用。以下按源码构建方式说明，复制现有应用的 pipeline，改以下字段：

| 字段 | 值 |
|---|---|
| Dockerfile | `deploy/Dockerfile` |
| Context | `.`（仓库根目录） |
| Repo | Artifactory 中的镜像路径，例如 `<artifactory>/<docker-local>/policy-compare` |
| Tag | `r<+pipeline.sequenceId>-<+codebase.shortCommitSha>` |

Build arguments：

| 参数 | 必填 | 值 |
|---|---|---|
| `ENGINE` | 是 | `mupdf`（或 `pdfjs`） |
| `RELEASE_ID` | 是 | 与 Tag 相同。必须是小写字母开头、只含小写字母/数字/连字符、≤40 字符 |
| `REVISION` | 是 | `<+codebase.commitSha>` |
| `CONTACT` | 否 | 页面“使用说明与版本信息”显示的维护渠道 |
| `NODE_IMAGE` | 否 | 默认 `artifactory.platform.axaxl-cloud.com/docker/library/node:24`（已验证可拉取） |
| `RUST_IMAGE` | 构建机不能拉 Docker Hub 时 | 默认是 Docker Hub 的 `rust:1.93.0-slim-bookworm@sha256:...`；可试 `artifactory.platform.axaxl-cloud.com/docker/library/rust:1.93.0-slim-bookworm`（未验证） |
| `NPM_REGISTRY` | 构建机不能直连 npmjs 时 | Artifactory npm 远程仓库 URL |
| `RUSTUP_DIST_SERVER` | `RUST_IMAGE` 不含 wasm32 目标且构建机不能直连 static.rust-lang.org 时 | 指向 `static.rust-lang.org` 的 Artifactory generic 远程仓库 |

Rust 依赖已经放在仓库的 `vendor/` 目录（`.cargo/config.toml` 指向它），构建时不访问 crates.io。更新 `Cargo.lock` 后，在能上网的机器上重新执行 `cargo vendor --locked --versioned-dirs vendor` 并提交。

不填可选参数时，使用上述默认镜像和公共 npm / rustup 源。

## CD 阶段：Kubernetes 服务

- **Manifest**：类型 `K8sManifest`，Git 连接器指向本仓库，Files `deploy/harness/templates`，Values `deploy/harness/values.yaml`。
- **Primary artifact**：Artifactory Docker 连接器，同一镜像路径，Tag `r<+pipeline.sequenceId>-<+codebase.shortCommitSha>`。
- **Infrastructure**：沿用现有 OpenShift 集群连接器和目标 namespace。
- **执行**：Rolling Deployment。

按环境覆盖 `values.yaml`：

- `host`：平台分配的内网域名；留空则由 OpenShift 在默认 apps 域名下生成。
- `routeLabels`：仅当内部入口控制器按标签选择 Route 时填写。
- `imagePullSecret`：namespace 默认服务账号不能从 Artifactory 拉取时填写已有凭据名。
- `networkPolicy.enabled`：namespace 默认拒绝入站且没有放行 openshift-ingress 的策略时设为 `true`。

## 部署后检查

```sh
oc -n <namespace> get pods -l app.kubernetes.io/name=policy-compare
oc -n <namespace> get route policy-compare -o jsonpath='{.spec.host}{"\n"}'
```

浏览器打开 `https://<route host>/`，会跳转到 `/releases/<RELEASE_ID>/`。页头应显示所选引擎。“使用说明与版本信息”中的版本：源码构建时等于本次 Tag，预构建时等于 `prebuilt/mupdf/build-info.json` 的 `releaseId`。点页面上的 **P02 Amount change** 示例做一次比较作为冒烟测试。

## 注意

- 每次构建的页面路径都包含新的 `RELEASE_ID`。Rolling 发布后，旧标签页中的用户需要刷新；需要新旧版本并存和可回滚的固定入口时，用 `docs/deployment.md` 的流程。
- 镜像以非 root 运行，静态文件全局可读，适配 OpenShift `restricted-v2` SCC 的随机 UID。
