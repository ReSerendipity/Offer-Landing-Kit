# 发布 / 回滚 规范（Offer-Landing-Kit 本地化）

> **来源与权威**：公共条款见 `ReSerendipity/.github` 的 `REPOSITORY_GUIDELINES.md` §10（发布 / 分发统一规范）。本文件是 Offer-Landing-Kit 的本地化落地，只写**本仓特有**内容，不重写、不抵触 §10 的公共条款。
> 核对基准：默认分支 `main`、`pushed_at=2026-10-12T05:14:52Z`、核对日期 2026-10-18。

---

## 0. 远端优先（根原则，不可改写）

发布链的构建、打包、扫描、上传、推送**全部在 GitHub Actions 远端完成**；本地只做三件事：改版本位、落 CHANGELOG、`git tag` + `git push`。本地不执行打包 / 镜像构建 / 发布上传。

本仓为纯静态单文件站点（`index.html` + `assets/`），**无本地构建步骤**，故天然满足该原则：本地不产出任何交付物，线上内容由 GitHub Pages 远端构建注入产生。

## 1. 版本号规范

- 遵循 **SemVer** `MAJOR.MINOR.PATCH`；预发布用 `-alpha.N` / `-beta.N` / `-rc.N`。
- **唯一权威版本位**：**无**。本仓根目录无 `package.json` / `pyproject.toml` / 配置文件版本字段（`git/trees/main` 实测确认），不存在机器可读版本位。
- 当前唯一的版本声明是 **README「Changelog」段的文本**：`v2.0 (2026-09-21)`、`v1.0 (2026-09-20)`。此文本即本仓当前的“版本位”。
- 必须同步的其它版本位：**无**（除 README「Changelog」段外，仓内无其它跟踪的版本串）。
- 版本位漂移校验入口：**无**。`tools/check-social-gap.mjs` 只核对断档测算口径，**不**校验版本号。
- 自动化方式：**手动（人工维护 README「Changelog」文本）**。本仓无 release-please（无 `.release-please-manifest.json`，无相关 workflow）。
- 与 §10.2「版本位须多点一致」的差距：本仓无机器版本位可钉，属**待收敛**项——待定（负责人 + 触发条件：本仓引入打包产物或独立版本声明文件后，再设权威版本位与一致性校验）。

## 2. CHANGELOG 管理

- 目标口径遵循 **Keep a Changelog**：`Added / Changed / Fixed / Deprecated / Removed / Security`；每次合并 `main` 在 `[Unreleased]` 追加；type 映射 `feat→Added`、`fix→Fixed`、`refactor→Changed`；发版时 `[Unreleased]` → `[VERSION] - YYYY-MM-DD`。
- **本仓现状（如实记录）**：**无 `CHANGELOG.md`**。变更记录写在 README「Changelog」段，采用 `vX.Y (YYYY-MM-DD)` 简写，**未采用** `[Unreleased]` 增量结构。
- 收敛方向：待定（负责人 + 触发条件：本仓转入多产物 / 频繁发版节奏、需要机器可聚合的变更记录时，再补 `CHANGELOG.md`）。**本次不创建** `CHANGELOG.md`。

## 3. 发布流程（远端）

- **本仓当前无发布链**：`.github/workflows/` 下**只有 `ci.yml` 一个文件**（`on: push branches:[main]` + `pull_request`，作业「离线口径核对（零依赖）」跑 `node tools/check-social-gap.mjs`）。**不存在** `release.yml`、`pages-deploy.yml` 等发布 workflow。
- **实际发布动作 = GitHub Pages 部署**：本仓 Pages 为 **legacy 分支源（branch `main`、path `/`）**，由 GitHub 注入的 `pages build and deployment` 构建（事件名 `dynamic`，非仓库内 workflow 文件）。**合并 / 推送到 `main` 即触发 Pages 重建并替换线上内容**——这一步就是本仓的“发布”。
- 因此本仓**没有** §10.2 的 `git tag` → 远端 draft Release → `published` 环节：实测 `tags=0`、`releases=0`，本仓无任何 tag / Release 对象。
- 与 §10.2 固定链的差距：**待收敛**——待定（负责人 + 触发条件：本仓产生需要版本化分发的产物时，再引入 tag + draft Release 流程）。
- 现网地址：`https://reserendipity.github.io/Offer-Landing-Kit/`（Pages `status=built`）。

## 4. 分发产物与完整性

| 产物 | 构建位置 | 标签 / 命名 | 完整性手段 |
|---|---|---|---|
| 静态单文件站点（`index.html` + `assets/*.png`） | **远端**（GitHub Pages 注入构建） | 无（本仓无 tag） | **无**（无 `SHA256SUMS.txt`、无签名；静态站无 Release 资产） |

- **不适用**：本仓无容器镜像、无 Python 包、无桌面 / APK 安装包，故不涉及 ghcr、Trivy、`provenance` / `sbom`、`pyproject.toml` 等条目。
- 完整性现状：本仓既无发布资产，也无校验清单；对外分发就是站点直链本身。
- **不使用 GPG 分离签名**（2026-10-18 裁决，本仓本就无此流程）。

## 5. 回滚

- **判定**（满足任一即触发）：以本仓 `docs/rollback.md` §1 为准——首页 404 / 样式丢失、社交预览指向错图、勾选状态“莫名丢失”（`localStorage` 前缀 `dzb2_` 按条目 id 存储，改 id 即丢）、`node tools/check-social-gap.mjs` 非 0 退出。
- **执行**：`git revert <bad-commit>`（保历史、可再 forward）；坏改动集中在 `index.html` 时可用 `git checkout <last-good-sha> -- index.html assets/` 单文件退回，再 `git push origin main`。完整步骤见 `docs/rollback.md` §2。
- **回滚单位是 commit，不是版本**：本仓**无版本锚点**（`tags=0`、`releases=0`），不存在可“回滚到 v1.0”的对象；README 写作的 `v2.0` / `v1.0` 无对应 tag 可指代（见 `docs/rollback.md` §0）。
- **本仓无“只提交不发布”的口子**：`git push origin main` 那一步才是发布动作；`[skip ci]` 挡不住 legacy Pages 的注入构建（判“这一笔到底发布没有”，看 `GET /repos/ReSerendipity/Offer-Landing-Kit/pages/builds/latest` 的 `commit` 是否为其）。
- **破坏性动作**：重指向 / 删除 tag、移动任何远端 ref → **须仓库所有者单独授权**（本仓当前无 tag；若引入 tag 依 §10.2 口径）。
- **回滚后验证**：`node tools/check-social-gap.mjs`（CI 里同一条，非 0 不算回滚完成）；再按 `docs/rollback.md` §3 四步人工走查：线上首页 hero 文案与 README 一致、链接预览卡图非 404、清 `dzb2_` 前缀后勾选能存住并刷新读回。

## 6. SLA / 错误预算

- 本仓为个人维护的纯静态单文件工具站，**无可用性监控、无健康端点、无运行手册**，因此**未定义**目标可用性与错误预算。
- **待定（负责人 + 触发条件）**：负责人 = 仓库所有者（ReSerendipity）；触发条件 = 本仓接入可用性 / 可用度监控后，再设定目标可用性并据此换算错误预算。**在此之前不设数值、不编造阈值。**

## 7. 发布前检查清单

> 本仓“发布”= 合并 / 推送到 `main` 触发 Pages 重建，故清单对应“合并前”检查。

- [ ] README「Changelog」段与本次发布内容一致（本仓唯一版本位，见 §1；无独立 `CHANGELOG.md`，属待收敛）
- [ ] `node tools/check-social-gap.mjs` 通过（CI 同一条口径核对命令，非 0 不得合并）
- [ ] lint / format：**无**（本仓无 lint / format 工具链，此项不适用）
- [ ] 改动 `index.html` 断档引擎时，已按 `docs/social-gap-check.md` 重跑用例表（C1/C2/C3/C6/C9）
- [ ] 涉及 `assets/*.png` 引用或链接时，已确认大小写与文件名一致（历史 404 成因）
- [ ] 镜像类仓项（Trivy / `provenance` / `sbom`）：**不适用**（本仓无镜像）
- [ ] **本地未产出任何发布交付物**（纯静态、无本地构建；线上内容全部远端 Pages 构建产生）

---

## 差异点采集清单（本仓取值，已逐项核实）

| 占位符 | 采集来源 | 本仓取值 |
|---|---|---|
| 版本位文件与字段 | 仓内版本声明 | **无**（无 `package.json` / `pyproject.toml` / 配置版本字段）；唯一文本版本声明在 README「Changelog」段（`v2.0`） |
| 其它需同步版本位 | 仓内版本串 | **无** |
| 版本一致性校验入口 | `tests/**` 等 | **无**（`tools/check-social-gap.mjs` 只管断档口径） |
| 自动化方式 | `.github/workflows/` | **手动**；无 release-please |
| 发布 workflow 文件名 | `.github/workflows/` | **无发布 workflow**；`workflows/` 仅有 `ci.yml` |
| 产物形态与标签规则 | 构建 workflow | 静态单文件站点；**无 tag** |
| 完整性手段 | Release 资产清单 | **无**（无 `SHA256SUMS.txt` / 签名；无 Release 资产） |
| 回滚判定阈值 | 运行手册 / 监控 | 见 `docs/rollback.md` §1（404 / 预览错图 / `dzb2_` 勾选丢失 / `check-social-gap.mjs` 非 0） |
| 回滚后验证命令 | smoke | `node tools/check-social-gap.mjs` + `docs/rollback.md` §3 四步 |
| 技术栈 | 声明文件 | 纯前端静态单文件（`index.html`，零依赖、无构建） |
| 部署目标 | Pages workflow | GitHub Pages（legacy 分支源 `main`；远端注入构建） |

## 禁止事项

- 不得重写 §10 的公共条款（远端优先、tag 驱动、版本位一致、不可变标签、扫描阻断、无 GPG）。
- 不得引入 `latest` 作发布基线；不得保留或新建 GPG 签名流程。
- 不得留空占位符，不得编造阈值——本仓无阈值 / 无监控的项一律写明“无”或“待定（负责人 + 触发条件）”。
- 不得为凑模板虚构发布 workflow 或分发产物（本仓确无发布链、确无打包产物）。
