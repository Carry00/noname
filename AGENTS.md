# AGENTS.md

个人 fork，来自 [adeFuLoDgu/noname](https://github.com/adeFuLoDgu/noname)（本身是 [libnoname/noname](https://github.com/libnoname/noname) 的活跃 fork）。线上部署：https://noname-adefulodgu.pages.dev （单机模式，不含联机对战服务器）。

## 项目结构

pnpm monorepo（`pnpm-workspace.yaml`）：
- `apps/core` — 浏览器前端本体，Vite + Vue，构建产物是纯静态站点
- `apps/mobile` / `apps/electron` — 移动端/桌面端壳
- `packages/server` — 联机对战服务，独立 Node 进程（单机模式不依赖它）
- `packages/fs` / `packages/jit` — 内部工具包

## 常用命令

```bash
pnpm install
pnpm build          # 根目录：依次构建 apps/core + server + 扩展包，合并输出到 dist/
pnpm dev            # 本地开发服务器
pnpm lint           # eslint
```

## 部署（Cloudflare Pages，已配置自动发布）

- Cloudflare Pages 项目：`noname-adefulodgu`
- GitHub Actions：`.github/workflows/deploy.yml`（上游自带，双发布 GitHub Pages + Cloudflare Pages）
- Secrets/Vars 已配置：`CLOUDFLARE_API_TOKEN`（scoped，仅 Pages Write）、`CLOUDFLARE_ACCOUNT_ID`、`CLOUDFLARE_PAGES_PROJECT_NAME=noname-adefulodgu`
- **发布流程**：改动文件 → `git commit`（**必须是真实文件改动，空提交不会触发 CI**——workflow 配了 `paths-ignore`，GitHub 对"空提交 + paths 过滤器"组合直接不触发，这是已知行为不是 bug）→ `git push origin master` → 几分钟后线上自动更新
- GitHub Pages 那个并行 job 目前会失败（仓库 Settings→Pages 未启用 "GitHub Actions" 来源），不影响 Cloudflare 部署，无需处理

## 已知坑 / 已修复的上游 bug

**importmap 路径与 node_modules→external 改名不同步**（本 fork 已修复，commit `7f2266da`）：

- `apps/core/scripts/build.ts` 里 rollup 的 `entryFileNames` 会把产物中包含 `node_modules` 的 chunk 文件名统一替换成 `external`（因为 Cloudflare `wrangler pages deploy` 默认过滤所有 `node_modules` 目录，必须改名才能上传）
- 但 `apps/core/scripts/vite-plugin-importmap.ts` 生成 importmap 时用的是 `require.resolve` 得到的**原始 node_modules 路径**，没有做同样的替换
- 结果：vue/pinyin-pro/dedent 等依赖的 importmap 路径指向不存在的文件，浏览器控制台 404，游戏 JS 崩溃打不开，**表现酷似"需要后端"，实际只是静态资源加载失败**（上游官方线上 `adefulodgu-noname.pages.dev` 也曾中招）
- 修复方式：在 `vite-plugin-importmap.ts` 的 `buildStart` 里给 `resolvedImportMap[key]` 加一个 `.replace(/node_modules/g, "external")`
- **同步上游改动时如果这段代码被覆盖，问题会复现**，需要重新打这个补丁

## Fork 维护

- 定期 `git fetch upstream && git merge upstream/master` 追更新（upstream = `adeFuLoDgu/noname`）
- 合并后重新检查上面那个 importmap 补丁是否还在
- Fork 仓库的 GitHub Actions 默认禁用，如果重新 fork 过一次记得去 Actions 标签页手动启用一次，启用之前的 push 不会补跑
