---
name: fanqie-novel-upload
version: 2.0.0
summary: 番茄小说自动上传新章节到作家后台（仅保存草稿，发布前人工确认）
description: |
  扫描本地章节 txt 文件，通过 CDP 连接 Edge 浏览器自动登录番茄小说作家后台，
  依次把新章节填到「创建章节」编辑器里并「存草稿」，最后由用户在 Edge 窗口中
  手动点击「下一步」→「发布」。用 ledger.json 台账防止重复上传，
  用 verify.js 校验并修复草稿的章节号/标题/正文缺失。
tags:
  - fanqie
  - 番茄小说
  - 自动上传
  - 小说发布
author: WorkBuddy
---

# fanqie-novel-upload

## 触发语

- 「传新章节」
- 「上传番茄小说章节」
- 「把新章节发到番茄」
- 「fanqie upload」

## 依赖

- Windows 系统，已安装 Microsoft Edge
- Node.js 22.22.2（WorkBuddy 托管版本已可用）
- **直接走 CDP（无需 agent-browser）**：脚本用 `cdp_helper.js` 连 9222 端口
- Edge 调试窗口已启动（端口 9222）

## 使用步骤

1. 确保 Edge 调试实例在运行（登录态长期保存在 `edge-profile`，无需重复扫码）：

   ```powershell
   C:\Users\Silen\WorkBuddy\fanqie-upload\start-edge.ps1
   ```

2. 把新章节文件放到：

   ```
   C:\Users\Silen\WorkBuddy\fanqie-upload\chapters\
   ```

   文件名格式（标题分隔符用**空格**，脚本会把 `_` 视为标题的一部分）：

   - `第62章 标题.txt`
   - `第62章.txt`

   正文若首行是章节标题（如「第六十二章 鬼面」），脚本会自动剔除，只取正文。

3. 运行上传脚本：

   ```powershell
   C:\Users\Silen\WorkBuddy\fanqie-upload\run.ps1
   ```

   或：

   ```bash
   cd C:\Users\Silen\WorkBuddy\fanqie-upload
   node upload.js
   ```

   每章约 10 秒，日志实时打印 `✓ 第N章《标题》已保存为草稿`。

4. **必做**：上传完成后运行校验脚本（平台偶发丢失章节号/标题，必须校验）：

   ```bash
   node verify_cdp.js --fix        # CDP 版复核 + 自动修复（✅ 本机可用，推荐）
   # node verify.js --fix         # ⚠️ 已失效：依赖 agent-browser 二进制，本机不存在，勿用
   ```

   `verify_cdp.js` 等价 `verify.js --fix`，但走纯 CDP（复用 `cdp_helper.js`），
   独立复核每篇草稿的 **章节号/标题/正文字数**（服务端字数≠本地即报异常），丢字段则自动补写并二次复核。
   用法：`node verify_cdp.js [--fix] [--from=N] [--to=M] [--volume=卷名]`，
   **等号与空格两种参数写法均支持**（`--from=136` 与 `--from 136` 等价）；
   批次复核建议带 `--volume` 圈定本批卷名，避免误把上一批未发布草稿也扫进来。
   **务必与 Edge 启动放在同一次调用内**（见下方「故障排除·WorkBuddy 环境硬坑」）。

5. 切到 Edge 窗口 →「作品管理 → 章节管理 → 草稿箱」，确认无误后点「下一步」→「发布」。

## 文件位置

| 文件 | 作用 |
|------|------|
| `C:\Users\Silen\WorkBuddy\fanqie-upload\config.json` | book_id、书名、章节目录 |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\ledger.json` | 已上传章节台账（含 draft_id） |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\cdp_helper.js` | 纯 CDP 客户端（连 Edge 9222，绕过 agent-browser） |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\upload_cdp.js` | **现用**核心上传脚本 v4（清存储→选卷→填→等已保存→存→**真值校验**），支持 `--force --from --to --repair --test` |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\upload.js` | 旧版（依赖 agent-browser，已弃用） |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\audit_content.js` | 全量内容审计：清缓存读服务端真值，比对标题/章节号/卷名/正文哈希，并检跨章重复 |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\audit_one.js` | 单章审计：`node audit_one.js <章号> [draftId]`（省略 id 则用 ledger 中的） |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\delete_drafts.js` | 按 draft_id 列表批量删除草稿，先 `--dry` 试运行核对定位再实删 |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\probe_draftlist3.js` | 导出草稿箱完整清单（自动翻页）到 `draftbox_list.json` |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\verify.js` | ⚠️ 草稿校验脚本（依赖 agent-browser 二进制，本机缺失，**已失效**） |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\verify_cdp.js` | ✅ **现用** CDP 版校验/修复脚本（等价 verify.js --fix，纯 CDP，本机可用） |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\start-edge.ps1` | 启动 Edge 调试窗口 |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\run.ps1` | 一键运行上传脚本 |
| `C:\Users\Silen\WorkBuddy\fanqie-upload\README.md` | 完整说明 |

## 关键实现要点（改脚本前必读）

1. **必须用 exe + 参数数组调用 agent-browser，绝不能走 shell**：
   `execFileSync(AB_BIN, args)`，其中
   `AB_BIN = %USERPROFILE%\.workbuddy\binaries\node\versions\22.22.2\node_modules\agent-browser\bin\agent-browser-win32-x64.exe`。
   用 `execSync(cmd, {shell:true})` 时，**正文里的换行符会被 cmd 截断**，只有第一段能填进去。

2. **正文编辑器是 ProseMirror**：直接写 `innerHTML` 不会更新字数统计（平台保存的仍是旧状态）。
   正确做法（纯 CDP）：`el.focus()` + `getSelection`+`Range.selectNodeContents` 程序化选中 → `Input.dispatchKeyEvent` 发 `Delete` 清空 → `Input.insertText` 注入正文。
   **鼠标点击 ProseMirror 聚焦不可靠**——填完标题后焦点停在标题 INPUT，插入文本会落到标题框，正文 length≈5；必须用 JS 程序化聚焦。注入后等 0.5s 校验 `.ProseMirror` 的 innerText 长度≥阈值。

3. **保存成功判定**：点「存草稿」后页面出现「已保存到云端」即成功（约 2 秒内）。
   若页面长期卡在「保存中」，说明该草稿页状态已污染，放弃它重新开新章节页。

4. **平台偶发丢字段**：批量上传时约 10%~20% 的草稿会丢失章节号或标题（正文仍正常），
   表现为草稿箱里显示「未命名草稿」或「第 章 xxx」。所以第 4 步的 `verify.js` 不能省。

5. **草稿箱每页 15 条**，超过需要翻页查看，别误判为上传失败。

6. **新建章节 URL**（`.../publish?enter_from=newchapter_0`）会恢复「当前活动未保存工作区」的 draft_id，
   而非每次都新建。若存在一个被污染/空的活动草稿，它会永远被复用，且编辑器加载即崩（选卷抛 `Uncaught`、正文注入 length≈4）。
7. **【关键修复】每次上传前必须清本地存储**：`CDP.call('Storage.clearDataForOrigin',{origin:'https://fanqienovel.com',storageTypes:'local_storage,indexeddb,websql'})`。
   清存储后 newchapter_0 不再复用坏死的本地活动指针，会开出**全新的空白工作区**（新 draft_id），编辑器恢复正常，选卷/填正文均可用。
   此操作不含 cookie，不会登出。这是 2026-09 攻关「坏草稿死锁（7681349808732242494）」的核心解法。

8. **【最易踩坑】校验前必须清本地缓存，否则读到的是缓存假象**：
   保存后整页 `reload` 读到的**仍可能是 localStorage 里的本地草稿缓存**，不是服务端真值。
   2026-09 的 v3 脚本正是因此产生**假阳性**——25 章里 21 章实际正文为空却全部「校验通过」。
   正确做法：`clearStorage()` → `navigate(/publish/{id})` → 等正文加载 → 再读。
   凡是「校验/审计」，都要先清缓存再读，否则结论不可信。

9. **【判定正文是否真的写进去】看字数统计和状态栏，不要只看 DOM**：
   - `.publish-header-count` = 正文字数；`.publish-maintain-info-status` = 状态（「保存中」→「已保存到云端」）。
   - **DOM 里有字 ≠ 编辑器 state 有字**。曾出现 ProseMirror 里 2125 字、但正文字数显示 0 的情况，
     此时点保存提交的是空正文。所以填充后必须轮询字数统计达标，且**等状态栏变成「已保存」再点存草稿**。

10. **【读取草稿页】正文是异步加载的，必须等字数出现**：
   打开 `/publish/{id}` 后只 `sleep(600)` 就读，会读到占位符「请输入正文」（去空白长度=5）而误判为空。
   必须 `waitFor` 到 `.publish-header-count > 0`（建议 12~20 秒超时）再读取。
   审计脚本曾因此把 16 章正常草稿误报为「内容为空且互相重复」。

11. **【修复空正文草稿】用 `--repair` 补正文，不要新建**：
   若草稿的卷名/标题/章节号都正确、只是正文为空，用
   `node upload_cdp.js --force --repair --from=95 --to=95 --test` 直接打开该草稿补正文，
   draft_id 不变、不产生新草稿。新建章节页（newchapter_0）的卷选择器 `.publish-header-volume-name`
   有时不渲染，会导致选卷步骤超时失败，故修复优先走 repair 模式。

13. **【上传前自动查重】新建模式每次都会产生新草稿**：
   `upload_cdp.js` 在上传前会自动打开草稿箱，按章节号比对，发现同章号草稿即**中止**并给出三条处置建议
   （改用 `--repair` 原地修复 / 先 `delete_drafts.js` 清理 / 加 `--allow-dup` 放行）。
   这是 2026-09「第81章存了 8 份草稿」事故的直接防线——**重试一次就多一份草稿**，切勿无脑重跑。
   参数：`--force`（忽略 ledger 重传）`--from=N --to=M`（范围）`--repair`（原地修）`--test`（只跑首章）`--allow-dup`（允许重复）。

12. **草稿箱入口与批量删除**：
   - 入口：`https://fanqienovel.com/main/writer/chapter-manage/{bookId}&{urlencode(书名)}`，进页面后还需点「草稿箱」标签
     （默认显示已发布章节；已发布表格与草稿表格各有一个分页器，翻页要定位表头含「修改时间」的那个表格）。
   - 删除：行内 `.icon-delete.tomato-delete` → 确认弹窗点「删除」。用 `delete_drafts.js`（先 `--dry`）按 id 精确删除，
     删除前务必先按章号核对，删除**不可逆**。

## 故障排除

- **编辑器崩溃 / 选卷抛 `Uncaught` / 正文注入 length≈4 / 反复复用同一坏草稿**：
  这是本地存储被污染导致 newchapter_0 死守一个损坏的活动工作区所致。**解法：每次上传前清本地存储**——
  在脚本 `attemptUpload` 开头调用 `CDP.call('Storage.clearDataForOrigin',{origin:'https://fanqienovel.com',storageTypes:'local_storage,indexeddb,websql'})`，
  重载后即开出全新空白工作区，选卷/填正文恢复正常（详见「关键实现要点」第 7 点）。`upload_cdp.js` 已内置此步骤。
- **agent-browser 找不到 Chromium**：本方案用系统 Edge + CDP，不需要下载 Chromium。若 Edge 没启动，运行 `start-edge.ps1`。
- **守护进程僵死（报错 `os error 10060` / `CDP command timed out`）**：agent-browser 守护进程不稳定，
  执行十几次操作后可能卡死。恢复步骤：

  ```powershell
  Stop-Process -Name "agent-browser-win32-x64" -Force -ErrorAction SilentlyContinue
  Stop-Process -Name "msedge" -Force -ErrorAction SilentlyContinue
  Start-Sleep -Seconds 4
  Remove-Item "$env:USERPROFILE\.agent-browser\default.pid","$env:USERPROFILE\.agent-browser\default.port","$env:USERPROFILE\.agent-browser\default.engine" -Force -ErrorAction SilentlyContinue
  Start-Process "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" -ArgumentList '--remote-debugging-port=9222','--user-data-dir=C:\Users\Silen\WorkBuddy\fanqie-upload\edge-profile','--no-first-run','--no-default-browser-check','--restore-last-session=false'
  ```

  然后 `agent-browser connect 9222`（首次常需重试 1~2 次）。上传脚本已内置重连重试。

- **登录态丢失**：重新运行 `start-edge.ps1`，扫码登录一次即可长期保持。

### WorkBuddy 环境硬坑（2026-09-07 实测，必读）

本环境跑此 Skill 有三个反直觉的坑，全部踩过并验证解法：

1. **Edge 在每次工具调用结束会被框架一并杀掉**——所以「启动 Edge」和「跑 Node 脚本」**必须放进同一次调用**（同一次 PowerShell/Bash 调用内 `Start-Process msedge` 后紧接着 `Start-Process node -Wait`）。分两次调用时，第一次调用里 Edge 看着起来了（端口 UP），但调用一结束进程就没了，Node 下次连必然 `ECONNREFUSED`。后台任务（`run_in_background`）是保活 Edge 的好办法，因为它会撑到 Node 跑完。

2. **托管 Node 被注入 `NODE_OPTIONS="--require=...node-language-shim.cjs"`**，这个 shim 强制把所有 HTTP 走 `127.0.0.1:52256` 代理，导致 Node 连不上本机 Edge 的 9222（即便 `curl`/PowerShell 能通）。**解法**：运行 node 前在同一进程内清掉注入与环境代理：
   ```powershell
   $env:NODE_OPTIONS=""; $env:HTTP_PROXY=""; $env:HTTPS_PROXY=""; $env:http_proxy=""; $env:https_proxy=""; $env:NO_PROXY="127.0.0.1,localhost"
   ```
   注意：用 `Start-Process -RedirectStandardOutput` 落盘 Node 输出最稳（PowerShell 的 `*> ` 重定向在此环境不可靠）。

3. **`edge-profile` 的 Session Storage 损坏会让 Edge 秒退**（端口一闪即灭、进程数 0）。**解法**：杀掉 msedge 后，删 `edge-profile\Default\` 下的损坏会话态文件（不动 Cookies，登录态可保住）：
   `Current Session` / `Current Tabs` / `Session Storage` / `Last Session` / `Last Tabs`，以及 `edge-profile\` 下的 `SingletonLock` / `SingletonCookie`。清完重启即可。全新临时 profile（`--user-data-dir=某临时目录`）能启动，可用来对照确认是 profile 损坏而非机器问题。

4. **PowerShell 的 `Start-Process` 可能因环境变量同时存在 `PATH` 与 `Path`（重复键）而崩溃**（报错「已添加项。字典中的关键字:"Path"所添加的关键字:"PATH"」）。**解法**：跑 Node 改用直接调用 `& $node args > out.txt 2> err.txt`（原生命令直调不重建环境字典，不触发此坑）；且本环境 PowerShell 工具**禁止调用 cmd.exe**。
   两个连带坑：① PS 的 `>` 重定向产出 UTF-16 编码文件（Read 工具读不了），先 `Get-Content file | Out-File out_utf8.txt -Encoding utf8` 转码再读；② 跑 node 前设 `[Console]::OutputEncoding=[System.Text.Encoding]::UTF8`，node 的 UTF-8 输出才不会变乱码。
- **重复上传**：`ledger.json` 按文件名防重；删除对应条目可强制重传。
- **草稿箱出现大量重复章节 / 内容为空 / 章节重复**：
  先 `node probe_draftlist3.js` 导出 `draftbox_list.json` 看全量家底（会自动翻页），
  再用 `node audit_content.js 81 105` 做真值审计（已内置清缓存+等异步加载，结论可信）。
  处置顺序：① 内容为空的章用 `--repair` 补正文；② 重复/空壳草稿用 `delete_drafts.js` 删（先 `--dry` 核对）。
  曾出现同一章存了 8 份草稿（第 81 章）的情况——多因上传重试与诊断探针「只填不存」产生，**每次重试都会留下一份草稿**。

- **审计报「内容为空」但草稿箱列表显示字数正常**：几乎一定是没等异步加载（见要点 10），
  或没清本地缓存（见要点 8）。先用 `node audit_one.js <章号>` 单章复测再下结论，别急着重传。

- **草稿箱堆积垃圾草稿**：草稿箱表格里点删除图标（`.tomato-delete`），确认弹窗点「删除」。

## 安全提示

番茄小说没有公开作者 API，本 Skill 使用浏览器自动化。请避免高频无人值守上传，
建议保持「脚本填草稿 + 校验 + 人工点发布」的半自动模式，降低账号风控风险。
