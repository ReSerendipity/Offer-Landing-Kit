# 回滚预案（rollback）

适用对象：本仓 `main` + GitHub Pages（**legacy 分支源**，构建源就是 `main`）。
证据基准：`de8f038c9e3e2197cad800e25a70335d2a1a6e0f`，核对日期 2026-10-12。

## 0. 先说清一件事：本仓没有版本锚点

`git/trees` 与 `/tags`、`/releases` 实测：tag **0 个**，release **0 个**。
README 的 Changelog 写了 `v2.0 (2026-09-21)` / `v1.0 (2026-09-20)`，但仓库里没有任何对象能被"回滚到 v1.0"指代。
所以本文件里的回滚单位只能是 **commit**。要按版本回滚，先补 tag（这是发布管理侧的待办，不在本文范围）。

## 1. 什么情况要回滚

按线上可见性排序，任一条命中即回滚，不要先就地热修：

| 症状 | 判定方式 | 已知成因 |
|---|---|---|
| 首页 404 / 样式丢失 | 打开 Pages 站点根路径 | 路径大小写（`de8f038c` 修的就是 og 图链接大小写导致的 404） |
| 社交预览指向错图 | 发消息给链接看卡片 | 同上，`assets/*.png` 引用与文件名大小写不一致 |
| 勾选状态"莫名丢失" | 改动清单条目 id 后刷新页面 | `README.md:51-53`：勾选按 **项目 id** 存 `localStorage`（前缀 `dzb2_`），id 一变，老勾选就找不到；这是数据模型问题不是发布问题，**回滚要连着回滚 id** |
| 社保口径数值变化 | `node tools/check-social-gap.mjs` 非 0 退出 | 该脚本是口径的唯一机器判据 |

## 2. 回滚步骤（首选 revert，其次 checkout 单文件）

```bash
# 1) 找到最后一个好的 commit（看 Pages 站点还能正常打开的那一次）
git log --oneline -10 main

# 2A) 整笔撤销（保留历史，首选）
git revert <bad-sha>

# 2B) 只把某个文件退回旧版（当坏改动集中在 index.html 时）
git checkout <last-good-sha> -- index.html assets/ 
git commit -m "revert: index.html 退回 <last-good-sha> 版本"

# 3) 推 main —— 这一推会自动触发 Pages 重建（无需手动发布）
git push origin main
```

**不要把 Pages 构建当成可选步骤**：实测每次 push 到 `main` 都会跑一次
`pages build and deployment`（GitHub 为 legacy Pages 注入的 workflow，事件名是 `dynamic` 而不是 `push`），
最近一次是 `#14`，commit `de8f038c`，成功。也就是说：**推上去就已经换掉了线上内容**，
回滚同理——`git push` 那一步才是"发布"动作，前面的 commit 只是准备。

如果你只想提交而**不想**换线上内容：`[skip ci]` 在本仓**做不到这件事**。实测于本文件的落地提交
`8597687a9a9cd533de133528de99a4c4b57229fa`：该 commit 的 workflow 运行数 `total_count=0`
（对照它的上一笔 `de8f038c` 是 2 条：`CI[push]` + `pages build and deployment[dynamic]`），
但 `GET /pages/builds/latest` 返回的仍是这个 commit，`created_at=2026-10-12T05:12:59Z`
（推送后 1 秒），`status=built` ⇒ **legacy Pages 的注入构建不吃 commit message**，推上去线上就换。

所以：本仓没有"只提交不发布"的口子。要留缓冲只能把改动放在分支上开 PR，合并到 `main` 才等于发布。
判"这一笔到底发布没有"，别看 Actions，看 `GET /repos/<owner>/<repo>/pages/builds/latest` 的 `commit` 是不是它。

## 3. 回滚后必须做的四步验证

```bash
node tools/check-social-gap.mjs        # ① 口径核对：CI 里跑的同一条，非 0 就不算回滚完成
```

② 打开线上首页，确认 hero 文案与 `README.md` 描述一致；
③ 把首页链接发到任意聊天窗口，确认预览卡片的图不是 404 占位；
④ 在浏览器里清掉 `localStorage` 前缀 `dzb2_` 再勾选一次清单，确认勾选能存住、刷新能读回
（这一步专门针对 §1 第 3 行的成因；如果回滚后这一条失败，说明回滚的 commit 选错了）。

## 4. 演练记录

回滚预案没演练过就等于没有。下表每次演练加一行；空表本身是要修的现状。

| 日期 | 演练场景 | 从发现到线上恢复 | 与本文件的偏差 |
|---|---|---|---|
| — | 未演练 | — | — |

> 建议第一次演练用"故意提交一个坏 og 图路径 → 观察 Pages 表现 → 按 §2 回滚"的方式做，
> 全程只在测试分支上做的话请另开分支，不要拿 `main` 练。
