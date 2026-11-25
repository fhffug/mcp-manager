# Branching

This repository uses two long-lived branches, in the shape projects such as CMake use:

- **`main`** holds the development line. Everything that ships passes through it.
- **`release`** holds the line that is published. A release is prepared on it, tagged there, and
  merged back into `main` so the two stay level.

There is no `develop`: `main` is where work lands, and CI on `main` has to be green.

## Short-lived branches

Cut from `main`, named `<type>/<slug>` where the type is one of the Conventional Commit types:

| Branch                                         | Cut from  | Opens a pull request against | Purpose                                                        |
| ---------------------------------------------- | --------- | ---------------------------- | -------------------------------------------------------------- |
| `feature/*`                                    | `main`    | `main`                       | A user-visible capability or a refactor                        |
| `fix/*`, `chore/*`, `docs/*`, `test/*`, `ci/*` | `main`    | `main`                       | A small correction of one kind                                 |
| `refactor/*`, `perf/*`                         | `main`    | `main`                       | Behaviour-preserving or speed work                             |
| `hotfix/*`                                     | `main`    | `main`                       | An urgent repair, also merged into `release`                   |
| `fix/*`, `chore/*`, `docs/*`, `test/*`, `ci/*` | `release` | `release`                    | A fix or the version bump that must reach the next release     |
| `refactor/*`, `perf/*`, `hotfix/*`             | `release` | `release`                    | The same, when the change is larger than a one-line correction |

Names are short and kebab-cased: `feature/probe-cache`, `fix/route-403`, `chore/release-0.2.0`.

## Merge rules

- Every merge is a **merge commit**, never a squash and never a rebase, so the branch keeps its own
  shape in the history and the pull request keeps a record:

  ```sh
  git switch main
  git merge --no-ff feature/probe-cache -m "Merge branch 'feature/probe-cache'"
  ```

- Never push straight to `main` or to `release`. Both are protected the same way: a pull request is
  required, the `Check` and `Commit messages` status contexts have to be green, and force pushes and
  deletions are refused. The `Check where the pull request comes from` job refuses a head branch whose
  name does not fit the table above.
- `release` takes fixes, the version bump, and back-merges from `main`; a feature reaches `release`
  only by way of `main`.

## Commits

Messages follow Conventional Commits, with a scope where it helps:

```text
feat(host): register the servers this bundle owns
fix(client): keep a masked secret out of the patch
test: cover the route refusals
ci: verify every push and pull request
docs: describe the three server sources
chore: scaffold the typescript package
```

Types in use: `feat`, `fix`, `test`, `ci`, `docs`, `chore`, `refactor`, `style`, `perf`, `revert`.
The summary is imperative and lower case, at most 100 characters, the body explains why the change is
needed, and one commit does one thing. The `Commit messages` job checks this on every push.

## Release

`release` is the published line; a version exists once the release branch carries its tag.

```sh
git switch release
git pull
git switch -c chore/release-0.2.0
npm version minor --no-git-tag-version   # or major; patch goes through the hotfix flow
# move the CHANGELOG entries from Unreleased to 0.2.0
git commit -am "chore(release): 0.2.0"
git push -u origin chore/release-0.2.0
```

Then:

1. Open `chore/release-0.2.0` against `release` and merge it with **Create a merge commit**, so CI
   proves the release line is green with the new version.
2. Tag it once it is final:
   `git tag -a v0.2.0 -m "v0.2.0" && git push origin v0.2.0`.
3. Pushing the tag runs the **Release** workflow: it builds the bundle, packs
   `dist/dsh-mcp-workbench-<version>.zip`, writes the notes from the matching `CHANGELOG` section, and
   publishes the GitHub release with that zip attached. Re-running the workflow by hand
   (`workflow_dispatch`, with the tag as its input) republishes or tops the release up. A tag with a
   suffix, such as `v0.2.0-rc.1`, is published as a pre-release: the workflow publishes it but leaves
   its label alone and does not mark it latest.
4. Open `release` against `main` and merge it with **Create a merge commit**. The tag is already an
   ancestor of `main` at that point.
5. Fixes found during a freeze land on `fix/*` branches and reach `release` by pull request. A patch
   release repeats steps 2–4 with `v0.2.1`.

A release is the only change that bumps `package.json`'s version or moves the `CHANGELOG` entries out
of `Unreleased`.

The same thing by hand, when the workflow cannot run:

```sh
npm run build
npm run pack                      # dist/dsh-mcp-workbench-<version>.zip, with its sha256 on stdout
node scripts/release-notes.mjs 0.2.0 --asset dsh-mcp-workbench-0.2.0.zip --sha256 <hash> --out dist/release-notes.md
```

## Hotfix

```sh
git switch main
git switch -c hotfix/route-403
# fix, test, commit
git push -u origin hotfix/route-403
```

1. Open `hotfix/route-403` against `main`, merge with a merge commit.
2. Tag `v0.2.1` on `main` — or, when the fix also ships the release line, on `release` after step 3.
3. Open the same branch against `release` and merge it there too, so the published line carries the fix.

## Forks and outside contributions

Fork the repository, clone the fork, and add the source repository as `upstream`:

```sh
git remote add upstream git@github.com:fhffug-code/mcp-manager.git
git fetch upstream
git switch -c fix/route-403 upstream/main
```

Push the branch to **your fork** and open the pull request against this repository's `main`, not
against a `feature/*` branch. Keep the branch rebased on `upstream/main`, run the full verification
loop first, and tick _Allow edits from maintainers_.
