# 通过 Harness 部署到 OpenShift

这条路径与公司其他应用一致：**Harness CI 构建镜像 → 推到 Artifactory → Harness CD（Kubernetes 部署）发布到 OpenShift**。它取代原 `deploy/kubernetes/policy-compare.yaml`。[docs/deployment.md](../../docs/deployment.md) 中的手工 `oc` 流程（多版本路径、固定入口切换、Playwright 验收）仍可用于正式验收，两者不要混用于同一项目。

应用本身只是一个 Node 静态文件服务器（端口 8080，`/healthz` 健康检查）。PDF 在用户浏览器里处理，没有后端接口、数据库或密钥。

## 目录

| 文件 | 用途 |
|---|---|
| `values.yaml` | Harness 解析 `<+artifacts.primary.image>`，以及副本、域名、拉取凭据等可覆盖参数 |
| `templates/policy-compare.yaml` | Deployment + Service + Route（edge TLS），可选 NetworkPolicy |

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

浏览器打开 `https://<route host>/`，会跳转到 `/releases/<RELEASE_ID>/`。页头应显示所选引擎，“使用说明与版本信息”中的版本应等于本次 Tag。用 `validation/synthetic-review-20261030/cases/P02` 做一次比较作为冒烟测试。

## 注意

- 每次构建的页面路径都包含新的 `RELEASE_ID`。Rolling 发布后，旧标签页中的用户需要刷新；需要新旧版本并存和可回滚的固定入口时，用 `docs/deployment.md` 的流程。
- 镜像以非 root 运行，静态文件全局可读，适配 OpenShift `restricted-v2` SCC 的随机 UID。
