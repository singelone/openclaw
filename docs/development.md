# OpenClaw 二次开发指南

面向在 OpenClaw 仓库做功能开发、调试的开发者：环境准备、构建、运行与日常命令。

## 环境要求

- **Node.js ≥ 22**（推荐 22 LTS）
- **pnpm**（推荐；也可用 bun，需与 lockfile 同步）
- Git 克隆仓库到本地

## 一、首次搭建（一次性）

在仓库根目录执行：

```bash
pnpm install
pnpm ui:build   # 首次会拉取并构建 Web/Control UI 依赖
pnpm build      # 编译 TypeScript → dist/
```

若 `pnpm install` 或 `pnpm ui:build` 报错 **`ERR_PNPM_PREPARE_PACKAGE` / Failed to prepare git-hosted package ... @tloncorp/api**（常见于 Windows 或网络受限环境），可先跳过依赖的 prepare 脚本再装：

```bash
pnpm install --ignore-scripts
pnpm ui:build
pnpm build
```

此时 Tlon 渠道插件可能不可用，其余功能正常；需要 Tlon 时可再尝试正常 `pnpm install` 或换网络/环境（如 WSL2）。

若 **`pnpm build`** 报 **`scripts/bundle-a2ui.sh: line 35: node: command not found`**：说明 `bundle-a2ui.sh` 在 WSL/Git Bash 里执行，而该环境没有 Node。**推荐做法**：在 WSL 内安装 Node 22 并全程在 WSL 里构建（在 WSL 终端中 `cd` 到仓库、`pnpm install --ignore-scripts`、`pnpm ui:build`、`pnpm build`）。若只在 Windows 上开发，可在 WSL 里安装 Node：`sudo apt update && curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - && sudo apt install -y nodejs`，再在 WSL 中执行上述命令。

之后如需跑 Gateway 并做渠道配对，可做一次引导（可选）：

```bash
pnpm openclaw onboard --install-daemon
```

未全局安装 `openclaw` 时，在仓库内用 `pnpm openclaw <子命令>` 即可。

## 二、日常怎么「运行」项目

OpenClaw 是 **CLI + Gateway 网关**：你改的是源码，运行的是编译后的 `dist/`。下面几种方式等价于「在 IDE 里点运行」。

### 1. 跑 Gateway 网关（最常用）

**前台运行，端口 18789：**

```bash
pnpm openclaw gateway --port 18789
```

或使用封装好的脚本（跳过渠道加载，启动更快，适合只调网关/控制面）：

```bash
pnpm gateway:dev
```

**改完代码自动重新编译并重启网关（开发环）：**

```bash
pnpm gateway:watch
```

监听 `src/`、`tsconfig.json`、`package.json`，有变更会重新 build 再起网关。

### 2. 跑 CLI 任意子命令

```bash
pnpm openclaw <子命令> [选项]
# 示例
pnpm openclaw status
pnpm openclaw channels status --probe
pnpm openclaw agent --message "hello" --thinking low
```

若 `dist/` 过期，`run-node.mjs` 会先触发一次 `pnpm build` 再执行。

### 3. 开发模式入口（不装服务、不常驻）

- `pnpm dev`：等同于 `pnpm openclaw`，把后面参数当 CLI 参数。
- `pnpm openclaw onboard`：只做引导，不安装 daemon（适合本机开发、不想动 systemd/launchd 时）。

## 三、在 VS Code / Cursor 里像「运行一个应用」一样用

没有内置 Spring Boot 那种一键「Run」，但可以当普通 Node 项目来跑：

1. **用终端**
   - 在集成终端里执行上面任意一条（如 `pnpm gateway:watch` 或 `pnpm openclaw gateway --port 18789`）。
   - 改代码后，若用 `gateway:watch` 会自动重建并重启。

2. **用调试 / 运行配置（可选）**
   - 在 `.vscode/launch.json` 里加一条 Node 启动配置：
     - **Program / 入口**：`${workspaceFolder}/scripts/run-node.mjs`
     - **参数**：`["gateway", "--port", "18789"]` 或 `["--dev", "gateway"]`（与 `gateway:dev` 一致）
     - **工作目录**：`${workspaceFolder}`
   - 需要「先构建再跑」时，可加 `preLaunchTask` 调用 `pnpm run build`（需在 `.vscode/tasks.json` 里定义对应 task）。
   - 这样即可用 F5 / 运行面板启动 Gateway，断点打在 `dist/` 里生成的 JS 上（若构建带 sourcemap，可映射回 `.ts`）。

3. **Windows 注意**
   - 若在 **PowerShell** 里跑，环境变量写法不同；`gateway:dev` 里的 `OPENCLAW_SKIP_CHANNELS=1` 在 Windows 下可写成：
     - PowerShell：`$env:OPENCLAW_SKIP_CHANNELS="1"; $env:CLAWDBOT_SKIP_CHANNELS="1"; pnpm run gateway:dev`
   - 推荐在 **WSL2 (Ubuntu)** 里做开发，与 CI/文档一致，见 [Windows (WSL2)](/platforms/windows)。

## 四、改代码后的流程

| 场景             | 建议命令 / 方式                                                                          |
| ---------------- | ---------------------------------------------------------------------------------------- |
| 只改 TS，跑网关  | `pnpm gateway:watch`（监听并自动 build + 重启）                                          |
| 改完想立刻测 CLI | `pnpm build` 后 `pnpm openclaw <子命令>`，或直接 `pnpm openclaw ...`（会自动按需 build） |
| 改完跑测试       | `pnpm test` 或 `pnpm test:fast`                                                          |
| 提交前检查       | `pnpm check`（格式 + 类型 + 静态检查），可选 `prek install`（与 CI 一致）                |

## 五、关键路径速查

- **源码**：`src/`（CLI 在 `src/cli`，Gateway 在 `src/gateway`，渠道在 `src/telegram`、`src/discord` 等）
- **构建产物**：`dist/`（入口 `dist/entry.js`，由 `openclaw.mjs` 加载）
- **脚本**：`scripts/run-node.mjs`（按需 build + 启动）、`scripts/watch-node.mjs`（监听 + 重启）
- **插件/扩展**：`extensions/*`，各自 `package.json`，改完在对应目录 `pnpm install` 即可

总结：**二次开发 = 拉仓库 → `pnpm install` → `pnpm ui:build` → `pnpm build`；日常运行用 `pnpm openclaw gateway` 或 `pnpm gateway:watch`，在 VS Code/Cursor 里用终端或自定义 launch 配置即可。**
