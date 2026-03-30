# Repository Guidelines

## 开发模式
DeerFlow 是一个全栈 monorepo。本仓库进行二次开发时，默认使用 Docker 开发模式，而不是宿主机本地 `make dev` 模式。除非用户明确要求切换到本地开发，否则遵循以下原则：

- 优先使用 `make docker-init`、`make docker-start`、`make docker-stop`、`make docker-logs`。
- 不默认要求宿主机安装 `nginx`；宿主机 `nginx` 只在本地开发链路 `make dev` 下是必需依赖。
- 不在未确认需求前擅自切换 `sandbox` 模式；`make docker-start` 会根据 `config.yaml` 自动判断 `local`、`aio` 或 `provisioner` 模式。
- 不启动长期后台服务作为“顺手验证”；只有在用户明确要求启动、联调或验证运行态时，才执行 `make docker-start`。

## 项目结构与修改入口
除非有特别说明，大多数工作流都应从仓库根目录执行。

- `backend/`：Python 3.12 服务端代码。
- `backend/app/`：Gateway、IM 渠道、应用层入口。涉及接口、控制层、服务编排时优先查看这里。
- `backend/packages/harness/`：可复用的 DeerFlow harness。涉及 agent、工具编排、执行框架时优先查看这里。
- `backend/tests/`：后端 pytest 测试。
- `frontend/`：Next.js 16 应用。
- `frontend/src/app/`：路由、页面和服务端入口。
- `frontend/src/components/`：可复用界面组件。
- `frontend/src/core/`：共享业务逻辑、客户端能力、状态与工具函数。
- `frontend/src/server/`：服务端逻辑、接口封装、服务端行为。
- `frontend/src/styles/`：全局样式与主题。
- `docker/`：Docker Compose、nginx 配置、容器化开发入口。
- `docker/docker-compose-dev.yaml`：Docker 开发编排。
- `docker/nginx/`：Docker 与本地开发使用的 nginx 配置。
- `scripts/`：根目录脚本，包含配置生成、Docker 启停、清理等辅助命令。
- `skills/`：运行时挂载到 sandbox 中的技能目录。

## Docker 二开工作流
进行二次开发时，默认遵循这条路径：

1. 在仓库根目录确认 `config.yaml` 已存在，且至少配置了一个可用模型。
2. 首次准备环境时执行 `make docker-init`。
3. 需要启动开发环境时执行 `make docker-start`。
4. 查看运行态日志时执行 `make docker-logs`、`make docker-logs-frontend`、`make docker-logs-gateway`。
5. 停止开发环境时执行 `make docker-stop`。

补充约束：

- `make docker-start` 是 Docker 开发的首选入口，不要自行拼接 `docker compose` 命令替代，除非是在排查脚本问题。
- 修改 `config.yaml` 后，如变更影响 `sandbox` 模式、模型配置或服务装配，应重新执行一次 `make docker-start` 验证。
- Docker 模式下，宿主机缺少 `nginx` 不构成阻塞；容器编排内已经包含 `nginx` 服务。
- 如果只是做静态代码修改、重构或补测试，不必每次都拉起整套容器；优先执行针对性的 lint、typecheck、build、pytest。

## 本地开发模式的边界
只有在以下情况，才考虑从 Docker 模式切到宿主机本地开发模式：

- 用户明确要求使用 `make dev`。
- 需要排查 Docker 特有问题，而问题在宿主机直接运行时更容易定位。
- 需要验证本地 `nginx` 代理、宿主机端口占用或非容器化运行行为。

若切换到本地开发，需要先满足这些前置条件：

- `make check` 通过。
- 宿主机安装了 `node`、`pnpm`、`uv`、`nginx`。
- 已执行 `make install`。

没有明确理由时，不要主动把用户从 Docker 模式带回本地模式。

## 配置与密钥约束
- 使用 `make config` 生成配置文件；如果配置结构升级，优先使用 `make config-upgrade`。
- 不要提交真实密钥。供应商 API Key 应保存在 `.env` 或 `config.yaml` 引用的环境变量中。
- 检查配置时，优先查看 `config.yaml` 是否缺失必要模型配置、`sandbox` 模式或关键路径，不主动读取含真实密钥的文件内容做展示。
- 如果 Docker 启动依赖配置文件缺失，优先通过仓库现有脚本或示例文件补齐，不手写替代文件结构。

## 构建、测试与验证命令
除非另有说明，以下命令均从仓库根目录运行。

- `make config`：根据示例文件生成 `config.yaml`、`.env` 和 `frontend/.env`。
- `make config-upgrade`：将新配置字段合并到现有 `config.yaml`。
- `make docker-init`：准备 Docker 开发环境。
- `make docker-start`：启动 Docker 开发环境。
- `make docker-stop`：停止 Docker 开发环境。
- `make docker-logs`：查看全部 Docker 开发日志。
- `make docker-logs-frontend`：查看前端容器日志。
- `make docker-logs-gateway`：查看网关容器日志。
- `make check`：检查宿主机本地开发依赖（`node`、`pnpm`、`uv`、`nginx`）；Docker 模式下通常不作为必经步骤。
- `make install`：安装宿主机本地开发依赖；仅在明确走本地开发链路时使用。
- `make dev`：启动宿主机本地开发环境；默认不作为二开首选。
- `cd backend && make lint`：运行 Ruff 检查并验证格式。
- `cd backend && make test`：运行后端 pytest 测试套件。
- `cd frontend && pnpm lint`：运行前端 lint。
- `cd frontend && pnpm typecheck`：运行前端类型检查。
- `cd frontend && pnpm build`：验证前端生产构建。

## 按改动类型选择验证策略
不要一律跑全量验证；先根据改动范围决定最小有效验证，再补充必要的集成验证。

- 仅改后端 Python 逻辑：至少执行 `cd backend && make lint` 与 `cd backend && make test`。
- 仅改前端页面、组件或前端逻辑：至少执行 `cd frontend && pnpm lint && pnpm typecheck && pnpm build`。
- 同时改前后端接口契约：分别执行后端测试与前端 lint/typecheck/build；如涉及联调，再补一次 `make docker-start` 验证运行态。
- 修改 Docker、nginx、启动脚本或 `config.yaml`：优先检查对应文件与脚本逻辑，再在需要时执行 `make docker-start` 和 `make docker-logs` 验证。
- 只改文档、注释或纯说明文本：无需跑全量测试，但应复查引用命令、路径和流程与仓库现状一致。

## 代码风格与命名约定
- Python：使用 4 空格缩进、双引号、公开 API 应补充类型标注，导入顺序按标准库 / 第三方 / 本地模块分组。Ruff 同时承担格式化和 lint。
- TypeScript/React：遵循 ESLint 与 Prettier 默认规则。组件使用 PascalCase，hooks 使用 `useX` 命名，共享逻辑放在 `src/core/*`。
- Python 测试与模块优先使用语义明确的 `snake_case`；前端文件命名遵循各目录现有风格。
- 修改现有模块时优先延续当前目录的代码组织方式，不为了“统一风格”顺手搬迁目录或重命名文件。

## 二次开发时的实现原则
- 先定位改动落点，再动代码；不要在不清楚责任边界时跨 `backend`、`frontend`、`docker` 大范围同时修改。
- 优先复用现有脚本、配置和项目约定，不引入平行的新启动方式或重复配置。
- 若需求只影响单一层级，避免把变更扩散到无关模块。
- 涉及接口、流式响应、反向代理或端口路由时，同时检查前端调用侧、Gateway、LangGraph 路由和 nginx 转发链路。
- 涉及 sandbox、subagent、skills 或工具编排时，优先阅读现有实现与配置，再决定是在 `backend/app/` 还是 `backend/packages/harness/` 修改。

## 测试规范
- 后端测试使用 `pytest`，位于 `backend/tests/`，通常命名为 `test_<feature>.py`。
- 每次修改后端行为时，都应新增或更新对应测试。
- 前端当前主要依赖 lint、typecheck 和 build；提交 UI 或交互变更前应全部执行。
- 如果修复的是运行时问题，优先补最小可复现测试，避免只改实现不补覆盖。

## Fork 与分支策略
本仓库是基于上游 DeerFlow 的 fork。默认采用“上游镜像主线 + 自定义开发主线”双轨模式，避免把二次开发历史和上游同步历史混在一起。

- `upstream/main`：上游官方主线，只读参考，不在本地直接开发。
- `origin/main`：fork 的镜像主线，默认目标是尽量与 `upstream/main` 保持一致。
- `custom/main`：当前 fork 的长期二次开发主线，默认所有定制化能力最终都合到这里。
- `feature/*`、`fix/*`、`chore/*`：日常开发分支，统一从 `custom/main` 切出，完成后再合回 `custom/main`。

默认约束：

- 不在 `main` 上直接做日常功能开发、Bug 修复或实验性提交。
- 需要实现需求时，优先确认当前分支是否为 `custom/main` 或其派生分支；如果不是，先切分支再改代码。
- `origin/main` 只允许承载极少数明确批准的 fork 维护型补丁，例如仅影响本地协作体验的 `.gitignore`、同步 fork 所需的元数据修正；这类改动应保持最小、独立、易识别。
- 不要把多个功能分支直接堆到 `main`，也不要把“同步上游”和“做自定义改动”放在同一个提交里。

推荐工作流：

1. 同步上游时，先更新 `upstream/main`，再让 `origin/main` 对齐上游。
2. 需要吸收上游最新能力到二次开发分支时，再把 `origin/main` 合并或 rebase 到 `custom/main`。
3. 做具体开发时，从 `custom/main` 切出 `feature/*` 或 `fix/*` 分支。
4. 功能完成并验证后，合回 `custom/main`，而不是直接合回 `main`。

常用命令参考：

- 查看 fork 主线相对上游是否有漂移：
  `git log --oneline upstream/main..origin/main`
- 查看当前二开分支相对 fork 主线新增了哪些提交：
  `git log --oneline origin/main..custom/main`
- 查看当前二开分支相对 fork 主线的代码差异：
  `git diff origin/main...custom/main`
- 从 `custom/main` 切新功能分支：
  `git switch custom/main && git switch -c feature/<topic>`

历史可读性要求：

- 优先让 `custom/main` 只承载“你自己的定制提交”，避免把上游 merge 噪音直接带进二开主线。
- 合并功能分支时，优先保持提交目的清晰；如果分支中有很多零散修补提交，可在合回 `custom/main` 前整理历史。
- 当需要区分“这是上游行为”还是“这是 fork 定制行为”时，默认以 `origin/main...custom/main` 为判断边界。

## 提交与 Pull Request 规范
- 遵循仓库已有的 Conventional Commit 风格，例如：`feat(gateway): ...`、`fix(config): ...`、`docs: ...`。
- scope 应尽量精确，对应实际变更区域，如 `frontend`、`sandbox`、`gateway`、`docker`、`config`。
- PR 需要说明：修改内容、修改原因、测试方式、关联 issue/PR 链接；涉及 UI 变更时附上截图。
- 若本次改动默认面向 Docker 二开场景，说明中应明确是否验证了 `make docker-start`、容器日志或运行态访问路径。

## 安全与环境注意事项
- 不要提交真实密钥、Cookie、认证文件或任何第三方服务凭据。
- 在 WSL2 或 Linux 环境下进行 Docker 开发时，优先确认 `docker info` 可用，再继续执行 Docker 相关命令。
- 不要在不了解后果的前提下直接修改 `docker/nginx/`、`docker-compose` 中的端口、卷挂载或网络名称；这些配置会影响整套开发环境。
- 如果命令失败，优先保留原始错误信息并说明阻塞点，不要用大范围重装或破坏性命令掩盖问题。
