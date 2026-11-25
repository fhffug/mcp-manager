# Changelog

All notable changes to this project are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- A **Release** workflow that runs when a `v*` tag is pushed — and on demand, with the tag as its input — and publishes the GitHub release itself: it builds the bundle, packs `dist/dsh-mcp-workbench-<version>.zip`, writes the notes from the matching `CHANGELOG` section and attaches the zip to the release.
- `npm run pack`, a portable `scripts/pack-release.mjs` that stages the files a profile needs and zips them on either Windows or the Linux runners, printing the file count, the size and the `sha256` of the archive.
- `scripts/release-notes.mjs`, which turns the changelog entry of one version into the body of its release.

### Changed

- The repository now has two long-lived branches, `main` and `release`, instead of cutting a `release/<minor>` branch per version. A release is prepared on `release` by a pull request, tagged there, and merged back into `main`; both branches require a pull request, the two status contexts, and refuse force pushes and deletions.
- The branch-source check follows the same model: `main` takes the contribution branches and the `release` line, and `release` takes fixes, chores and a back-merge from `main`.

### Fixed

- A release the workflow writes is published even when `gh` leaves it as a draft, and the release it tops up is refreshed deliberately: the publishing step now ends with `gh release edit … --draft=false`, so neither a draft `gh` created nor one a person drafted by hand stays hidden. That last step marks the release latest unless it is a pre-release — a tag such as `v0.2.0-rc.1` keeps the label it was given and is not forced into `latest`.

## [0.1.0] - 2026-11-25

### Added

- An **MCP** entry in the sidebar, after **Plugins** and the plugin marketplace, opening an **MCP management** page that lists the three sources of servers: the static `@deepseek-ai/dsh-mcp-client` rows of the profile (read-only), the servers of the profile's `@deepseek-ai/dsh-mcp-manager` row (editable), and the servers other plugins registered on `mcpManager` (read-only).
- The `managed` source as the one place this plugin writes: a server of that row's `servers` map can be edited, switched on or off, restarted and removed, because `enabled` is an ordinary field of the map.
- Adding a server to that map from the page, and deleting one from the form footer after a confirmation, both through `configEditor.edit()` with the rest of the row preserved.
- Switching a server between `streamable-http` and `stdio` in its form, in edit mode as well as when adding: both halves keep their own draft while the form is open, and saving writes the half the selector points at and drops the other.
- A test call that prefills its arguments from the tool's input schema and shows the answer as JSON, as a foldable tree, or as the raw request and response.
- **Reset** beside **Call**, which puts the prefilled arguments back.
- Each tool of each server keeps the arguments typed for it, so leaving a tool and coming back — or clicking the same row again — keeps the text instead of overwriting it.
- The page's bands can be resized: drag one of the three separators, or focus it and use the arrow keys; a double click puts that band back where it started, and the shares are remembered in `localStorage`.
- Search boxes over the server list and the tool list, and tool rows that keep their description to two lines.
- Read-only rows that still probe and still take test calls, with the reason they cannot be edited shown on the row.
- Probing a server over its own MCP session: tool names, descriptions, input schemas, timings, a name and schema budget, and a staged failure report. The session is opened on first use, reused by later probes and test calls, and closed when the server is written, restarted, or the plugin stops.
- A `mcp_workbench` tool with `list`, `detail`, `probe`, `call`, `save`, `add`, `remove`, `setEnabled` and `restart`; every change needs `confirm: true`.
- Every word of the page in the eight languages the Harness offers — Chinese, English, Japanese, Spanish, French, German, Russian and Portuguese — following **Settings → General → Language** and switching as soon as another one is picked. The dictionaries are typed against the English one, so a forgotten key fails `npm run typecheck`, and `locale/<id>.json` carries the bundle metadata for the same eight.
- A Chinese manual, [README.zh.md](README.zh.md), beside the English [README.md](README.md), each linking to the other.
- `POST /api/mcp-workbench`, a same-origin JSON endpoint, as the only channel between the page and the Host.
- A vitest suite of 210 tests, including a dependency-free stdio MCP fixture server that the probe and the call path exercise for real, a unit suite over the connection pool, and a jsdom test of the page's switch and its bands.
- GitHub Actions: the verification pipeline, a check that a pull request comes from a branch of the right kind, and a Conventional Commit check.

### Changed

- The page is one nested set of bands rather than three side-by-side cards: the servers take a fifth, then the tools three tenths and the selected tool the last three tenths, separated by hairlines that double as drag handles; the server search stays in the server band and the tool search sits beside the test button.
- The band called **Try a call** is now **Call**, in both languages.
- The MCP page moved out of Settings → Plugins into its own sidebar entry; the settings tab is gone, and the bundle now injects the sidebar and layout client packages instead of the settings page.
- The default `maxToolSchemas` is 200 rather than 64, so a server with a large toolset still reports every input schema.
- The repository is organised around `main` and `release/<minor>`, with every contribution landing on `main` through a pull request and a merge commit; there is no `develop` branch.
- The plugin no longer keeps a server set of its own: the `servers` key of its own config is gone and a profile that still sets it gets a warning instead. Every mutation now lands on the profile's `@deepseek-ai/dsh-mcp-manager` row, which is the only row this plugin writes.

### Fixed

- The switch on a server row looked inert: the click was ignored until the list was re-read, and a re-read taken before the profile write landed put the old state back. The row now shows the new state at once, reconciles with the profile afterwards, and reports a refusal under the row.
- A switch the plugin manager could not apply was reported as done and then reverted two seconds later, because the manager answers `overridden` for a row the profile patch inserts onto itself. Such a row is now offered as read-only, with that reason on the row, instead of a switch that cannot work.
- A profile row inserted by the patch reported empty `env` and `headers`, because its raw config arrives in the inherited layer while the override stays empty; saving such a row would have written the config without its environment.
- The page used theme tokens that do not exist, so its text, borders, cards and buttons kept the light palette in dark mode; every colour now comes from a real `--dsw-alias-*` token.
- Test connection and test calls opened a new MCP session, and a new process, on every request: a streamable HTTP server saw one session per click. One session per server is now kept and reused, and a call whose session died is reported instead of being replayed.
- The test suite is type-checked: `tsconfig.test.json` covers `src/` and `tests/`, `npm run typecheck` runs it, and the type errors the IDE was reporting (a mock logger, a textually typed status table, an untyped mock tuple) are fixed.
- The argument list of a `stdio` server lost its quoting when the form read and wrote it, so a path with a space came back as several arguments. A quoted argument now survives the round trip, and a list that was already split that way is joined again.
- The probe summary of a tool list asked for two words the dictionaries did not have, and nothing showed how many tools were returned once the list was cut short. Both are words now, and the summary says when the list was truncated.
- A test call whose argument box did not hold a JSON object printed an English sentence in every language. The page now asks for one in its own words, and the parser reports why it refused instead of returning a sentence.

[Unreleased]: https://github.com/fhffug/mcp-manager/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/fhffug/mcp-manager/releases/tag/v0.1.0
