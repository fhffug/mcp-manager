# AGENTS.md

How to work in this repository when you are an agent rather than a person.

## 1. What this repository is / 这个仓库是什么

**English.** `@local/dsh-mcp-workbench` is a DeepSeek Harness plugin, written in TypeScript and run on
Node 22. It is a private bundle installed into a profile, not a package published to a registry. It owns
one entry in the sidebar — **MCP**, after **Plugins** and the marketplace — and the page that entry
opens: servers on the left, tools in the middle, one tool's detail and test call on the right. The same
operations are exposed to the model through the `mcp_workbench` tool. Both surfaces call one service,
and one same-origin route (`POST /api/mcp-workbench`) carries the browser traffic. The three server
sources are `profile` (static `@deepseek-ai/dsh-mcp-client` rows of the profile patch, read-only here),
`managed` (the `servers` map of a profile row whose name is `@deepseek-ai/dsh-mcp-manager`, the only
place this plugin writes) and `manager` (whatever else registered on `mcpManager`, read-only). The page
speaks the eight languages the Harness ships with; the manual is [README.md](README.md) in English and
[README.zh.md](README.zh.md) in Chinese, and the two stay in step.

**中文。** `@local/dsh-mcp-workbench` 是一个用 TypeScript 编写、运行在 Node 22 上的 DeepSeek Harness
插件。它是装进 profile 的私有 bundle，不发布到任何 registry。它在左侧栏占一个入口「MCP」（排在【插件】
与插件市场之后），并拥有它打开的页面：左边服务器、中间工具、右边单个工具的说明与试调用。同样的操作也
通过 `mcp_workbench` 工具暴露给模型。两个界面共用同一个 service，浏览器只走一条同源路由
`POST /api/mcp-workbench`。三个来源分别是 `profile`（profile 补丁里的 `@deepseek-ai/dsh-mcp-client`
静态行，这里只读）、`managed`（profile 里名字是 `@deepseek-ai/dsh-mcp-manager` 的那一行的 `servers`
映射，也是本插件唯一写入的地方）和 `manager`（其它插件注册到 `mcpManager` 的，只读）。页面支持 Harness
自带的八种语言；手册是英文 [README.md](README.md) 与中文 [README.zh.md](README.zh.md)，两份要同步改动。

## 2. How to run commands on this machine / 这台机器怎么跑命令

**English.** The Harness shell cannot run inside this workspace: the folder is owned by
`Administrators` and the agent runs unelevated, so `SetNamedSecurityInfoW` fails with `grantWrite(...)`
and every `pwsh` call is refused. Do not work around it by asking for an elevated sandbox. Use the
WebStorm MCP server instead, which is already attached to this project:

- File work: `read_file`, `create_new_file`, `apply_patch`, `reformat_file`, `optimize_imports`,
  `lint_files`, `get_file_problems`, `search_text`, `search_regex`, `list_directory_tree`, `git_status`.
- Shell work: `execute_terminal_command` with `projectPath: E:/Project/DeepSeek Harness MCP manager`,
  **one command per call** (the tool treats `;` as a separator and corrupts compound commands).
- `npm` is not on `PATH` there: call `npm.cmd`. Node is on `PATH`; node_modules binaries are reached as
  `node node_modules/<tool>/bin/<entry>` — do not use `npx`, it hangs here.

**中文。** Harness 自带的 shell 在这个工作区里跑不起来：目录属主是 `Administrators`，而 agent 以普通
权限运行，`SetNamedSecurityInfoW` 报 `grantWrite(...)` 失败，所有 `pwsh` 调用都会被拒绝。不要靠申请
提权沙箱绕过。请改用已挂在本项目上的 WebStorm MCP 服务器：读写代码用 `read_file`、`create_new_file`、
`apply_patch`、`reformat_file`、`optimize_imports`、`lint_files`、`get_file_problems`、`search_text`、
`search_regex`、`list_directory_tree`、`git_status`；跑命令用 `execute_terminal_command`，`projectPath`
填 `E:/Project/DeepSeek Harness MCP manager`，**一次只跑一条命令**（该工具把 `;` 当分隔符，会把复合命令
弄坏）。那里没有 `npm`，用 `npm.cmd`；Node 在 `PATH` 上，node_modules 里的工具用
`node node_modules/<tool>/bin/<entry>` 调用，不要用 `npx`（会挂住）。

## 3. The verification loop / 验证循环

**English.** Run these from the repository root, in this order, and fix what they report before you
commit. CI runs the same commands on Node 22.

```sh
node node_modules/typescript/bin/tsc -p tsconfig.json --noEmit   # host
node node_modules/typescript/bin/tsc -p tsconfig.client.json     # client
node node_modules/typescript/bin/tsc -p tsconfig.test.json       # tests
node node_modules/eslint/bin/eslint.js "src/**/*.{ts,tsx}" "tests/**/*.{ts,tsx}" "scripts/**/*.mjs"
node node_modules/prettier/bin/prettier.cjs --check .
node node_modules/vitest/vitest.mjs run --coverage
node node_modules/typescript/bin/tsc -p tsconfig.json && node scripts/build-client.mjs
node scripts/verify-bundle.mjs
node scripts/check-commits.mjs <base-sha> [head-sha]           # Conventional Commits
```

`npm.cmd run typecheck`, `lint`, `format:check`, `test`, `build` and `verify:bundle` are the same
things packaged up; prefer them when you only need the aggregate answer. Coverage must stay at or
above 70% of lines.

**中文。** 在仓库根目录按顺序执行，提交前把报告的问题修掉；CI 在 Node 22 上跑同一套命令。
`npm.cmd run typecheck｜lint｜format:check｜test｜build｜verify:bundle` 是它们的打包版，只想要结论时用
这些即可。行覆盖率不得低于 70%。`node scripts/check-commits.mjs <base> [head]` 校验提交信息。

## 4. Install and reload the bundle / 安装与重载

**English.** Use the `plugin_manager` tool, never a hand-edited profile:

- Install or update: `action: install_bundle`, `target: "E:/Project/DeepSeek Harness MCP manager"`.
- Force a reload after a rebuild: `action: remove_bundle`, `target: "@local/dsh-mcp-workbench"`, then
  install again. Editing a dependency that is still linked only says `already-installed`.
- A moved directory needs the re-install; the answer then carries `application: restart-required` with
  `restartReason: package-moved`, and the model-visible tool schema only returns after a restart.
- Verify with the `mcp_workbench` tool: `list`, then `probe <name>`, then `call <name>` with
  `confirm: true`. A settings page that never renders usually means the Host half threw during apply.

**中文。** 一律用 `plugin_manager`，不要手改 profile。安装/更新：`action: install_bundle`，
`target: "E:/Project/DeepSeek Harness MCP manager"`。重建后要强制重载：先 `remove_bundle`
`@local/dsh-mcp-workbench`，再装一次；只改已链接的依赖会得到 `already-installed`。目录搬过家必须重装，
返回里会有 `application: restart-required` 与 `restartReason: package-moved`，重启后工具 schema 才回来。
用 `mcp_workbench` 工具自检：`list` → `probe <名字>` → `call <名字>`（要 `confirm: true`）。页面打不开
通常是 Host 端在 apply 时抛了异常。

## 5. Branches, pull requests, commits / 分支、PR、提交

**English.** Two long-lived branches: `main` (the development line) and `release` (the published
line). Work happens on `<type>/<slug>` cut from `main`, with the type taken from the Conventional
Commit it carries: `feature/`, `fix/`, `chore/`, `docs/`, `test/`, `ci/`, `refactor/`, `perf/`.
Every change lands by pull request, merged with a **merge commit** — never squash, never rebase — so the
branch keeps its own shape. Never push straight to `main` or `release`: both require a pull request,
the two status contexts, and refuse force pushes and deletions. The required status contexts are
`Check` and `Commit messages`; `Check` gates on `Verify on Node 22`, the branch-source guard and the
commit-message check. Commit messages are Conventional Commits, lower case, imperative, at most 100
characters, one thing per commit. Details: [docs/GITFLOW.md](docs/GITFLOW.md).

**中文。** 只有两条长期分支：`main`（开发主线）和 `release`（发布线）。日常从 `main` 切
`<type>/<slug>`，类型就用对应提交的前缀：`feature/`、`fix/`、`chore/`、`docs/`、`test/`、`ci/`、
`refactor/`、`perf/`。所有改动都走 PR，并且用 **merge commit** 合并，禁止 squash、禁止 rebase，保留
分支自己的形状。永远不要直接 push 到 `main` 或 `release`：两者都要求 PR、两个必需检查，并拒绝强推与
删除。必需检查是 `Check` 与 `Commit messages`，其中 `Check` 汇总 `Verify on Node 22`、来源分支检查和
提交信息检查。提交信息用 Conventional Commits，小写、祈使句、不超过 100 字符、一个提交只做一件事。
细则见 [docs/GITFLOW.md](docs/GITFLOW.md)。

## 6. Release checklist / 发版清单

**English.**

1. `git switch release && git pull`, then `git switch -c chore/release-<version>`.
2. Bump the version: `npm version minor --no-git-tag-version` (or `major`); a `patch` release goes
   through the hotfix flow instead.
3. Move the `[Unreleased]` entries of `CHANGELOG.md` under `## [<version>] - <yyyy-mm-dd>`.
4. `git commit -am "chore(release): <version>"`, push the branch, open it against `release`, and let CI
   go green; merge with a merge commit.
5. Collect freeze fixes on `fix/*` branches, each with a pull request into `release`.
6. `git tag -a v<version> -m "v<version>"` on `release`, then `git push origin v<version>`. The **Release**
   workflow builds the zip, writes the notes from the changelog and publishes the GitHub release; rerun
   it by hand (`workflow_dispatch`, the tag as input) to top a release up. A tag with a suffix, such as
   `v<version>-rc.1`, is published as a pre-release: the workflow publishes it but keeps its label and
   never marks it latest.
7. Open `release` against `main`, merge with **Create a merge commit**, so the two lines stay level.
8. Keep `release`: a later patch release repeats steps 2–7.

**中文。**

1. `git switch release && git pull`，再 `git switch -c chore/release-<版本>`。
2. 改版本号：`npm version minor --no-git-tag-version`（或 `major`）；只改 patch 的版本走 hotfix 流程。
3. 把 `CHANGELOG.md` 里 `[Unreleased]` 的内容移到 `## [<版本>] - <yyyy-mm-dd>`。
4. `git commit -am "chore(release): <版本>"`，推分支、对 `release` 开 PR，等 CI 全绿后用 merge commit 合入。
5. 冻结期的修复走 `fix/*` 分支，各自向 `release` 开 PR。
6. 在 `release` 上 `git tag -a v<版本> -m "v<版本>"`，再 `git push origin v<版本>`。**Release** workflow
   会自动 build、打包、按 changelog 生成说明并发布 GitHub release；要补发就手动跑一次
   （`workflow_dispatch`，输入标签）。带后缀的标签（例如 `v<版本>-rc.1`）会作为预发布版发布：workflow
   只发布它，保留它原有的标签，绝不把它标成 latest。
7. `release` 对 `main` 开 PR，用 **Create a merge commit** 合并，让两条线保持一致。
8. `release` 保留；之后的补丁版重复第 2–7 步。

## 7. Hotfix checklist / 热修清单

**English.** `git switch main && git switch -c hotfix/<slug>`; fix, verify, commit; PR into `main`
with a merge commit; then PR the same branch into `release` so the published line carries the fix too,
and tag `v<version>` where the fix ships.

**中文。** `git switch main && git switch -c hotfix/<slug>`；修复、验证、提交；对 `main` 开 PR 并 merge
commit 合入；再把同一分支对 `release` 开 PR 合入，让发布线也带上这个修复，并在实际发布的那条线上打
`v<版本>` 标签。

## 8. Code rules / 代码规则

**English.**

- **No `@deepseek-ai/*` dependency.** The bundle resolves those from the running installation. Contracts
  are hand-written in `src/types/dsh.ts`; extend that file instead of importing a dsh package.
- **Never write the profile config by hand.** Read with `ctx.configEditor.entries()` /
  `configuration()`, write with `edit()`. Keep `{ __jsExpr }` values intact.
- **Write only on the manager row.** Every mutation — add, save, switch, remove — lands in the
  `servers` map of the profile row named `@deepseek-ai/dsh-mcp-manager`, through `writeManagerRow()`.
  This plugin owns no server set of its own, and it never touches a static `mcp-client` row.
- **No secret in a browser payload.** `src/edit.ts` masks a value whose key looks like a secret; the
  client gets a presence marker. An omitted field keeps its value, `null` deletes it.
- **One operation, one method.** A capability is a method on `Workbench`, then a route action, then a
  tool row, then a control. The route, the tool and the component hold no logic.
- **One session per server.** `src/connection.ts` owns the client and the transport of every server the
  plugin talks to: `probe` and `call` go through `ConnectionPool.use()`, never through a connection of
  their own, and every write, restart and shutdown closes the session it invalidated.
- **Every failure has a code.** `WorkbenchFailure` with a `FailureCode`; the route maps it to a status,
  the tool renders `<code>: <message>`.
- **Style only with real tokens.** Use `--dsw-alias-*` and `--ds-font-family-code`; an invented token
  silently falls back and breaks dark mode.
- **One id for a panel.** The row registers into `sidebar.panellist` and the page into the keyed `main`
  slot with the same value; a mismatch throws when a person picks the row. The page itself pads its top
  by `--dsh-frame-top-clearance`, because it is not the conversation.
- **The client does not import Harness client packages.** `react` arrives through `require('react')`;
  the bundle is wrapped by `scripts/build-client.mjs`.
- **Every visible word comes from a dictionary.** `src/client/locales.ts` is the only place prose lives:
  `en` is the shape, `LocaleKey` is derived from it, and each of the other seven languages is a
  `Record<LocaleKey, string>`, so a forgotten key fails `npm run typecheck` instead of showing blank.
  A helper that renders words takes `t` as an argument (`probeHeadline()` does) rather than importing
  the dictionary. `locale/<id>.json` carries the same eight ids for the bundle metadata.
- **Tests come with the change.** New action → its refusal paths; parser change → the value that broke.

**中文。**

- **不要依赖 `@deepseek-ai/*`**：bundle 从运行中的安装里解析它们。契约手写在 `src/types/dsh.ts`，需要新
  接口时扩这个文件，而不是 import dsh 包。
- **绝不手写 profile 配置**：读用 `ctx.configEditor.entries()` / `configuration()`，写用 `edit()`；
  `{ __jsExpr }` 值必须原样回写。
- **只写 manager 行**：增、改、启停、删都只落在名字是 `@deepseek-ai/dsh-mcp-manager` 的那一行的
  `servers` 映射里，一律经 `writeManagerRow()`。本插件不拥有自己的服务器集合，也从不碰静态
  `mcp-client` 行。
- **浏览器侧不出现秘密**：`src/edit.ts` 会把 key 像秘密的值遮起来，客户端只拿到存在标记。字段省略表示
  保持原值，`null` 表示删除。
- **一个操作一个方法**：能力先落在 `Workbench` 的方法，然后是路由动作、工具参数行、页面控件；路由、工具和
  组件里不放逻辑。
- **一个服务器一个会话**：`src/connection.ts` 持有插件要谈话的每个服务器的 client 与 transport；`probe`
  与 `call` 都经 `ConnectionPool.use()`，绝不自己新建连接；任何写入、重启、停机都要关掉它作废的那个会话。
- **每个失败都带 code**：`WorkbenchFailure` + `FailureCode`，路由映射成 HTTP 状态，工具渲染成
  `<code>: <message>`。
- **只用真实 token 写样式**：用 `--dsw-alias-*` 和 `--ds-font-family-code`；编造的 token 会静默回退，
  深色模式就会坏。
- **一个面板一个 id**：侧栏行注册进 `sidebar.panellist`，页面注册进 keyed `main`，两处的值必须相同；
  不一致时用户一点那一行就抛错。页面顶部要按 `--dsh-frame-top-clearance` 留白，因为它不是会话页。
- **客户端不 import Harness 的客户端包**：`react` 通过 `require('react')` 进来，bundle 由
  `scripts/build-client.mjs` 包裹。
- **所有可见文案都来自字典**：`src/client/locales.ts` 是唯一放文案的地方——`en` 定形状，`LocaleKey` 由它
  推导，其余七种语言都是 `Record<LocaleKey, string>`，漏键在 `npm run typecheck` 就报错而不是显示空白。
  会渲染文案的辅助函数把 `t` 当参数传进去（`probeHeadline()` 就是这么做的），不要直接 import 字典。
  bundle 元数据 `locale/<id>.json` 用同样的八个 id。
- **测试随改动一起提交**：新动作要有拒绝路径，解析器改动要有当初坏掉的那个值。

## 9. Traps that bite / 会咬人的坑

**English.**

- The five MCP rows live inside one `- insert:` group of the profile patch, so `configuration().override`
  is empty and the raw config arrives in `inherited`. `rawConfigOf()` handles it; a write to such a row
  appends a new top-level override row, and removing the last change removes that row again.
- `configEditor.edit()` can change a row's `config` only: it cannot insert or delete a row, and it cannot
  touch `disabled`. A static `mcp-client` row is therefore read-only here: the plugin manager cannot
  override a row its own patch layer inserts (it answers `overridden`), so this plugin does not offer a
  switch for those rows at all — it shows the reason instead. Probe and call still work on them.
- A `managed` server lives in the `servers` map of a profile row named
  `@deepseek-ai/dsh-mcp-manager`; there `enabled` is an ordinary field, so switching it is an ordinary
  config write. Moving a statically declared `mcp-client` row into that map is the only way to switch a
  server the profile already declares — a plugin cannot delete or re-enable an `insert:` row. Keep
  `!!js` out of that map: its schema holds `env`/`headers` to strings, so `configEditor.edit()` fails
  validation on the marker; `expressionPath()` refuses the write first and names the path. A secret
  there belongs in the credential store as `${credential:<name>}`.
- `install_bundle` can fail once with `EPERM … rename package.json.<digits>`; an identical retry succeeds.
- A live session is not replayed. `ConnectionPool.use()` drops a dead session and retries only when
  `retryOnDisconnect` is set: `probe` sets it, a `tools/call` does not, because the server may already
  have run the call. Close the session of any server you rewrote — the service already does.
- esbuild's postinstall is blocked under the bundled npm, but its JS API works — `scripts/build-client.mjs`
  uses it. Never call `npx esbuild`.
- git prints to stderr, which shows up red in the terminal tool; that is not a failure. One command per
  call, or `;` corrupts the command line.
- The stdio fixture prints `'dsh-no-such-command-anywhere' 不是内部或外部命令…` during the
  transport-failure test; that line is expected.
- The GitHub token here cannot merge a pull request (`403 Resource not accessible by personal access
token`), so a human presses **Merge**; `gh` is not installed.

**中文。**

- profile 补丁里五个 MCP 行都在同一个 `- insert:` 组里，所以 `configuration().override` 是空的，真实原始
  配置出现在 `inherited` 里。`rawConfigOf()` 已经处理；对这种行写入会在文件末尾追加一条顶层覆盖行，撤掉
  最后一次改动时那条行会再次被删掉。
- `configEditor.edit()` 只能改某行的 `config`：不能插入/删除行，也碰不到 `disabled`。静态 `mcp-client`
  行因此在这里只读：plugin manager 覆盖不了自己那一层插入的行（会答 `overridden`），所以本插件干脆不给
  这些行开关，只说明原因；探测与调用照旧可用。
- `install_bundle` 偶尔第一次失败于 `EPERM … rename package.json.<数字>`，原样重试即成功。
- 自带的 npm 屏蔽了 esbuild 的 postinstall，但它的 JS API 可用，`scripts/build-client.mjs` 用的就是它。
  不要调 `npx esbuild`。
- git 的提示走 stderr，在终端工具里是红色，不是失败。一次一条命令，否则 `;` 会把命令行弄坏。
- 传输失败的测试里，stdio 夹具会打印 `'dsh-no-such-command-anywhere' 不是内部或外部命令…`，这是预期的。
- 这里的 GitHub token 没有合并 PR 的权限（`403 Resource not accessible by personal access token`），
  合并要人点 **Merge**；`gh` 没装。

## 10. Definition of done / 完成的标准

**English.** A change is done when: the full verification loop above is green, including coverage and
`verify:bundle`; a test covers the new behaviour and its refusal path; the commit messages pass
`scripts/check-commits.mjs`; the changelog has a line if a person can notice the change; the docs that
describe the touched surface (README table, `docs/GITFLOW.md`, this file) still match the code; the
branch is pushed and a pull request against `main` is open with the two required checks green; and — if
the bundle runs in a profile — it was reinstalled and the `mcp_workbench` tool answered `list`.

**中文。** 一次改动算完成，需要：上面整套验证全绿（含覆盖率与 `verify:bundle`）；新行为及其拒绝路径都有
测试；提交信息通过 `scripts/check-commits.mjs`；用户能感知的改动在 CHANGELOG 里有一行；描述被改动面的
文档（README 的表格、`docs/GITFLOW.md`、本文件）仍然和代码一致；分支已推送、面向 `main` 的 PR 已就绪且
两个必需检查全绿；如果 bundle 要在 profile 里跑，已经重装过并用 `mcp_workbench` 的 `list` 验证通过。
