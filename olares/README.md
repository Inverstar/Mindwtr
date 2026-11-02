# Olares 三合一镜像（mindwtr-merged）

把上游两个官方镜像 + npm 上的 MCP helper 拼成一个镜像，供 Olares 上的 Mindwtr 应用使用。

```
ghcr.io/inverstar/mindwtr-merged:<上游版本>
```

> 本目录只做「拼装」，**不编译源码**：内容全部来自上游已发布的
> `ghcr.io/dongdongbh/mindwtr-cloud` 和 `ghcr.io/dongdongbh/mindwtr-app`，
> 加上 npm 包 `mindwtr-mcp`。所以上游发版后几分钟就能出一个新镜像。

---

## 目录内容

| 文件 | 作用 |
|---|---|
| `Dockerfile` | 三阶段拼装：cloud 提供 `/app` 源码与 bun，app 提供 nginx + PWA，npm 装 MCP helper |
| `start.sh` | 容器内进程编排：Cloud → 等 `/ready` → MCP helper → 等 `/healthz` → nginx |
| `nginx.conf` | 单端口分流：`/mcp`、`/healthz` → helper；`/health`、`/ready`、`/v1/*` → Cloud；其余 → PWA |
| `patch-cloud.py` | 构建期给 Cloud 的 `server.ts` 注入 `/mcp` 转发分支（幂等） |
| `compose.yaml` | 本机冒烟测试用的参考 compose |

## 容器内拓扑

| 进程 | 监听 | 对外 |
|---|---|---|
| `bun` → Cloud | `127.0.0.1:8787` | 否 |
| `node` → MCP helper | `127.0.0.1:8723` | 否 |
| `nginx` → 统一入口 | `0.0.0.0:8080` | **是** |

---

## 构建

### GitHub Actions（推荐）

workflow：`.github/workflows/olares-image.yml`

**只有手动触发**（`workflow_dispatch`），没有 push / 定时。

理由：Olares 那边的 chart 本来就要手动提 PR 才能升级，镜像先自动构建出来没有任何
收益，只会堆出一批没人处理的 run。什么时候构建、构建哪个版本，由你决定。

**版本自动跟随**，三个输入都留空就是「全用最新」：

| 输入 | 留空时的行为 |
|---|---|
| `upstream_version` | `gh api repos/dongdongbh/Mindwtr/releases/latest` → 当前 `1.3.7` |
| `mcp_version` | `registry.npmjs.org/mindwtr-mcp/latest` → 当前 `1.1.13` |
| `tag_suffix` | 不加后缀，tag 就是上游版本号 |

镜像本身永远由上游 `mindwtr-cloud:<版本>` + `mindwtr-app:<版本>` 拼装（`pull: true`
强制重新解析基础镜像），MCP helper 用 npm 上最新的 `mindwtr-mcp`，所以不存在
「镜像里还是旧代码」的情况。构建日志里会打印实际用到的三个版本。

> **分支约定：`main` 与上游 `dongdongbh/Mindwtr` 完全一致，不放任何本仓改动。**
> 所有构建文件只活在 `olares-app` 分支上。
>
> 这样能跑起来，靠的是下面这条官方规则
> （[events-that-trigger-workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)）：
>
> - `workflow_dispatch`：「This event will only trigger a workflow run if the workflow
>   file **exists on the default branch**.」——文件不在默认分支时它**不会被注册**，
>   第一次 dispatch 会 404（实测：`workflow ... not found on the default branch`）。
> - 但紧接着一句：「Once a workflow has **run at least once**, you can dispatch it
>   against **any branch or tag** via the GitHub API or GitHub CLI.」——只要跑过一次，
>   之后 `gh workflow run --ref <分支>` 就能对着任意分支跑，文件在不在默认分支都无所谓。
> - `schedule`：「Scheduled workflows **will only run on the default branch**.」
>   ——定时任务跑的是默认分支的最新提交，这条无法绕过。
>
> 所以本仓不依赖默认分支，也不用 push 触发器：workflow 已经跑过（注册完成），
> 之后一律用带 `ref` 的 CLI 触发：
>
> ```bash
> gh workflow run olares-image.yml --ref olares-app
> ```
>
> **万一哪天 dispatch 报 404**（workflow 记录被清掉），恢复办法是把
> `olares-image.yml` 里的 push 触发器临时加回来，往 `olares-app` 推一次任意改动
> 让它跑起来完成注册，然后再删掉。文件注释里留了可直接取消注释的片段。

## 上游发新版后的完整流程

```bash
# 1. 出镜像（版本自动取上游最新 release，约 8 分钟）
cd C:/Night/github.com/Inverstar/Mindwtr
gh workflow run olares-image.yml --ref olares-app
gh run watch $(gh run list --workflow=olares-image.yml --limit 1 --json databaseId --jq '.[0].databaseId')

# 2. 确认镜像已推到 GHCR（tag 就是上游版本号，例如 1.3.8）
#    页面：https://github.com/Inverstar?tab=packages&repo_name=Mindwtr

# 3. 去 apps 仓改 chart：values.yaml 的 image.tag、Chart.yaml 的 version 加一位
cd C:/Night/github.com/Inverstar/apps

# 4. 提 PR（标题必须符合 bot 格式，版本号必须严格大于 main 上的）
gh pr create --base main --head feature/mindwtr \
  --title "[UPDATE][mindwtr][1.1.6] 升级镜像到 1.3.8"
```

第 3 步之后记得重新打包 chart zip 上传 Olares，`imagePullPolicy: IfNotPresent`
意味着 tag 没变就不会重新拉镜像。

### 输入速查

| 场景 | 输入 |
|---|---|
| 跟上游最新版（最常用） | 全部留空 |
| 指定上游版本 | `upstream_version=1.3.8` |
| 指定 MCP 版本 | `mcp_version=1.1.14` |
| 同一版本要重建（改了 Dockerfile） | `tag_suffix=r2`，得到 `1.3.8-r2` |
| 覆盖已存在的 tag | `force=true` |

```bash
gh workflow run olares-image.yml --ref olares-app                        # 跟最新
gh workflow run olares-image.yml --ref olares-app -f upstream_version=1.3.8
gh workflow run olares-image.yml --ref olares-app -f tag_suffix=r3      # 同版本重建
```

**为什么重建必须换 tag**：Olares 的 Deployment 用 `imagePullPolicy: IfNotPresent`，
沿用旧 tag 时节点不会重新拉取，新镜像根本不会生效。

### 本机手动构建

```bash
cd olares

docker buildx build \
  --platform linux/amd64 \
  --load \
  --build-arg CLOUD_IMAGE=ghcr.io/dongdongbh/mindwtr-cloud:1.3.7 \
  --build-arg APP_IMAGE=ghcr.io/dongdongbh/mindwtr-app:1.3.7 \
  --build-arg MCP_VERSION=1.1.13 \
  -t mindwtr-merged:test \
  .
```

构建成功的标志（缺这行说明 patch 失败）：

```
BUILD: Cloud /mcp passthrough patch applied.
[patch-cloud] PATCH OK: 已在 .../server.ts 插入 /mcp 与 /healthz 转发分支。
```

### 本机冒烟测试

```bash
CLOUD_TOKEN=$(openssl rand -hex 32)
MCP_TOKEN=$(openssl rand -hex 32)

docker run -d --name mindwtr-test -p 8080:8080 \
  -e MINDWTR_CLOUD_AUTH_TOKENS="$CLOUD_TOKEN" \
  -e MINDWTR_MCP_HTTP_TOKEN="$MCP_TOKEN" \
  -e MINDWTR_MCP_CLOUD_TOKEN="$CLOUD_TOKEN" \
  mindwtr-merged:test

sleep 15
curl -s http://localhost:8080/health   # ok
curl -s http://localhost:8080/ready    # ok
curl -s http://localhost:8080/healthz  # ok

# 必须输出 /app，否则 bun --filter 会 ENOENT
docker exec mindwtr-test sh -c 'pwd; ls -l /app/package.json'

# 末尾应出现「全部就绪，对外入口端口 8080」
docker logs mindwtr-test 2>&1 | tail -20

docker rm -f mindwtr-test
```

---

## ⚠️ 历史遗留：GHCR 上的 `1.3.7` 是修复前的旧镜像

`ghcr.io/inverstar/mindwtr-merged:1.3.7` 是 2026-11-02 在 apps 仓构建的**有 bug 的版本**
（缺 `WORKDIR /app`，容器起来就 `error: ENOENT`）。修复后的镜像是 `1.3.7-r2`。

所以：

- chart 的 `image.tag` 现在必须是 `1.3.7-r2`，**不要回退到 `1.3.7`**
- 本 workflow 的「tag 已存在就跳过」判断对 `1.3.7` 也成立，不会自动修好它
- 要修的话，手动跑一次：`upstream_version=1.3.7`、`tag_suffix=r2` 之外，
  把 `tag_suffix` 留空并勾 `force`，用新构建覆盖 `1.3.7`
- 上游发到 1.3.8 之后就不存在这个问题了，直接用 `1.3.8`

## 镜像名与仓库归属

`ghcr.io/inverstar/mindwtr-merged` 里的 `inverstar` 是 GitHub **账号名**，
不是仓库名。任何属于这个账号、带 `packages: write` 的仓库都能推同一个镜像名。

所以这个构建从 `Inverstar/apps` 搬到 `Inverstar/Mindwtr` 之后：

- chart 里 `repository: ghcr.io/inverstar/mindwtr-merged` **不需要改**
- GHCR Package 页面可能仍显示关联到旧仓库，属 UI 展示，不影响 pull
- 需要的话可在 Package 设置 → Manage repository 里切换关联

---

## 环境变量（运行时注入，不进镜像层）

| 变量 | 谁校验 | 规则 |
|---|---|---|
| `MINDWTR_CLOUD_AUTH_TOKENS` | Cloud | 逗号分隔；每个 20-512 字符，字符集 `[A-Za-z0-9._~+/=-]` |
| `MINDWTR_MCP_HTTP_TOKEN` | MCP helper | ≥ 16 字符 |
| `MINDWTR_MCP_CLOUD_TOKEN` | Cloud | 必须是上面某个 token |
| `MINDWTR_CLOUD_CORS_ORIGIN` | Cloud | `NODE_ENV=production` 下必填，start.sh 有兜底默认值 |

**数据按 token 分命名空间**：Cloud 用 `sha256(token)` 当文件名（`<hash>.json`），
换 token 等于换一个空数据集。迁移数据的正确做法是把旧 `<hash>.json`
复制成新 token 的 `<hash>.json`，不要去猜旧 token。

---

## 已修复的两个启动阻断（2026-11-02）

| 症状 | 根因 | 修复位置 |
|---|---|---|
| 容器起来几毫秒就 `error: ENOENT` 退出 | 最终阶段 `FROM app AS merged` 基于 nginx 基底，没有 `WORKDIR`，cwd 是 `/`；`bun run --filter` 找不到 `/app/package.json` | Dockerfile 加 `WORKDIR /app`；start.sh 再 `cd /app` 兜底 |
| Cloud 导入期抛 `MINDWTR_CLOUD_CORS_ORIGIN must be set in production.` | 镜像 `ENV NODE_ENV=production`，`apps/cloud/src/server-config.ts` 在 production 下强制要求显式 origin | start.sh 用 `: "${MINDWTR_CLOUD_CORS_ORIGIN:=http://localhost:5173}"` 兜底；chart 侧传真实应用域名 |

---

## 发版之后要做什么

镜像推好了，Olares 不会自动升级，还得改 chart：

1. `Inverstar/apps` → `mindwtr/values.yaml`：`image.tag` 改成新 tag
2. `mindwtr/Chart.yaml`：`version` 加一位（如 1.1.5 → 1.1.6）
3. 提 PR，标题必须符合 bot 格式：`[UPDATE][mindwtr][1.1.6] 说明`
   - 版本号必须**严格大于** main 上的版本
   - 被 bot 关掉的 PR 不能重开，要新建
4. 重新打包 chart zip 上传 Olares
