---
name: sap-fe-release
description: |
  Step-by-step workflow for executing sap.fe patch releases and recovering from release failures.
  Use this skill whenever the user mentions: release process, version bump, pre-release commit,
  post-release commit, xmake Release job, contributorRelease_sap.fe, SNAPSHOT cleanup,
  version change only, release branch, or anything about stabilizing/restoring SNAPSHOT versions.
  Also use it for diagnosing broken releases, wrong versions after failed runs, or UI5 SNAPSHOT
  dependency errors in the release Artifactory repo.
---

# sap.fe Patch Release Skill

**Scope**: patch releases only (not Patch Set 0 / minor versions).  
**Official wiki (follow this for the happy path)**: https://wiki.one.int.sap/wiki/spaces/fioritech/pages/6398493417/Release+SAP.FE

---

## Key Concepts (read this first if you are new)

| Term | What it means |
|---|---|
| **Gerrit** | SAP's internal code review tool (like GitHub Pull Requests). All code changes go through Gerrit before merging. URL: https://git.wdf.sap.corp/q/project:sap.fe |
| **xmake** | SAP's internal build and CI system (like Jenkins). Runs automated builds, tests, and deployments. |
| **SNAPSHOT** | A development/unstable version, e.g. `1.136.5-SNAPSHOT`. It will change. |
| **Stable version** | A released version, e.g. `1.136.4`. Fixed and immutable — what customers receive. |
| **VERSION CHANGE ONLY** | A commit that only updates version numbers, no code changes. xmake creates these automatically during the release. |
| **Voter** | An automated Jenkins job triggered by Gerrit that builds the change and posts V+1 (pass) or V-1 (fail). |
| **CR+2** | Code Review +2 = two people approved the Gerrit change. Required before merging. |
| **V+1** | Verified +1 = the automated build (voter) passed. Required before merging. |
| **Artifactory** | SAP's internal artifact repository — where built JARs are stored and downloaded from during builds. |
| **Renovate** | An automated bot that creates dependency bump changes (Maven, YAML, `@ui5/cli`) — but NOT npm packages. |

---

## Quick Reference Links

| Resource | URL |
|---|---|
| Release wiki (happy path steps) | https://wiki.one.int.sap/wiki/spaces/fioritech/pages/6398493417/Release+SAP.FE |
| Release job (**first choice**) | https://ui5infra.only.sap/job/contributorRelease_sap.fe/ |
| Release job (xmake fallback) | https://prod-build10300.xmake.tools.sap/job/default_1/job/sap.fe-Release/ |
| xmake build job (nightly CI) | https://prod-build10300.xmake.tools.sap/job/default_1/job/sap.fe-build-linuxx86_64/ |
| Renovate → Gerrit converter job | https://ui5infra.only.sap/view/sap.fe/job/sap.fe_renovateToGerrit/ |
| Renovate PRs (GitHub) | https://github.tools.sap/ux/sap.fe/pulls |
| Gerrit sap.fe changes | https://git.wdf.sap.corp/q/project:sap.fe |
| GBT Notes upstream pom.xml | https://github.tools.sap/AP-FIORI-CORE/nw.core.gbt.notes.lib.reuse/blob/main/pom.xml#L21 |
| FiorAI reuse upstream pom.xml | https://github.tools.sap/AP-FIORI-CORE/ux.eng.fioriai.reuse/blob/main/pom.xml#L22 |

---

## Happy Path

Follow the steps in the wiki above in order. The links in the wiki go directly to the right pages and the steps work as described the vast majority of the time.

**What a successful release produces** — exactly three VERSION CHANGE ONLY commits on the branch:
```
1. VERSION CHANGE ONLY  ← UI5 snapshot bump (from sapui5 bot)
2. VERSION CHANGE ONLY  ← Pre-release: removes all -SNAPSHOTs, stabilizes versions
   └── xmake build runs here and deploys the release artifacts
3. VERSION CHANGE ONLY  ← Post-release: adds -SNAPSHOT back, bumps to next version
```

**Why the version rolls back by 1**: the SNAPSHOT is always the *next planned* version. So `1.X.44-SNAPSHOT` means "we are building towards 1.X.44" — but the version being released *right now* is `1.X.43`. The build subtracts 1 to get the released version.

---

## Recovery Workflow (When the Release Job Fails)

Use this only if the xmake Release job did not complete successfully.

### Step 1 — Diagnose first

Open a terminal in your sap.fe repo and run:
```bash
git fetch origin
git log --oneline origin/rel-1.X | head -5
grep '"version"' packages/sap.fe.core/package.json
```

Work through this checklist:

```
□ Are there TWO VERSION CHANGE ONLY commits since the last code commit?
  YES → both pre and post-release ran. Check if the Slack notification came.
  NO (only one) → post-release didn't run. Read the xmake job console log.

□ Does the xmake log show "Could not resolve ... SNAPSHOT"?
  → A -SNAPSHOT version is still in one of the Maven POMs.
  → Go to "Manual Pre-Release Commit" below.

□ Are packages/*/ui5.yaml files corrupted (version: true)?
  Run: grep -r "version: true" packages/*/ui5.yaml
  → See Known Failure Mode #1 below.

□ Is the current branch version one higher than expected?
  e.g. shows 1.X.44-SNAPSHOT but you expected 1.X.43-SNAPSHOT
  → Post-release ran despite a build failure. Target release version is 1.X.43 (current minus 1).
  → Confirm with the DevOps architect before proceeding.
```

### Step 2 — Manual Pre-Release Commit

Only create this manually if the automated job could not.

**A — Set up a worktree** (an isolated copy of the repo for your branch)
```bash
git worktree add .claude/worktrees/FixRelease-rel-1.X origin/rel-1.X
cd .claude/worktrees/FixRelease-rel-1.X
git fetch origin && git reset --hard origin/rel-1.X
```

**B — Determine the release version**
```bash
grep '"version"' packages/sap.fe.core/package.json
# Shows e.g. "1.120.44-SNAPSHOT" → release version is 1.120.43 (current SNAPSHOT minus 1)
```

**C — Stabilize all SNAPSHOT versions**

Replace sap.fe version in all ~29 files:
```bash
find . -name "pom.xml" -o -name "package.json" | xargs grep -l "1.120.44-SNAPSHOT" \
  | xargs sed -i '' 's/1\.120\.44-SNAPSHOT/1.120.43/g'
sed -i '' 's/1\.120\.44-SNAPSHOT/1.120.43/g' lib/docbuilder/src/sap-ui-version.json
```

Replace UI5 parent POM version (`maven/sap-fe-parent/pom.xml`):
```bash
sed -i '' 's/1\.120\.51-SNAPSHOT/1.120.50/g' maven/sap-fe-parent/pom.xml
```

Replace dependency versions in `utils/pom.xml` AND `maven/sap.fe.templates/pom.xml`:
```bash
sed -i '' 's/1\.120\.51-SNAPSHOT/1.120.50/g' utils/pom.xml
sed -i '' 's/1\.120\.1-SNAPSHOT/1.120.0/g' utils/pom.xml
# ...repeat for each SNAPSHOT that appears
# NEVER touch com.sap.zen entries
```

**D — Verify nothing is missed**
```bash
# Should return ZERO results (only com.sap.zen is allowed to stay as SNAPSHOT)
grep -r "SNAPSHOT" maven/ utils/ --include="*.xml" | grep -v "com.sap.zen"
```

**E — Commit and push to Gerrit**
```bash
git add -p                     # review each file as you stage it
git status                     # final check before committing
git commit -m "[INTERNAL][VERSION CHANGE ONLY] Release 1.120.43" --no-verify
git push origin HEAD:refs/for/rel-1.X
```

If the push fails with `missing Change-Id in message footer`:
```bash
CHANGE_ID="I$(git rev-parse HEAD | openssl dgst -sha1 | awk '{print $2}')"
git commit --amend -m "$(git log -1 --format='%B')

Change-Id: $CHANGE_ID" --no-verify
git push origin HEAD:refs/for/rel-1.X
```

**F — Get approvals then re-trigger xmake**
1. Open the Gerrit change: https://git.wdf.sap.corp/q/project:sap.fe
2. Get **CR+2** (two people must review and approve) and wait for **V+1** (automated build passes).
3. Once the change is merged, go back to the wiki and trigger the xmake Release job again.

---

## Known Failure Modes

### 1. YAML Corruption: `version: true`

**Symptom**: A `packages/*/ui5.yaml` contains `version: true #FrameworkVersion#` instead of an actual version string like `1.120.51-SNAPSHOT`.  
**Cause**: The version-bump automation has a YAML serialization bug.  
**Impact**: The xmake voter fails on the VERSION CHANGE ONLY commit; the build is blocked.

**How to spot it**:
```bash
grep -r "version: true" packages/*/ui5.yaml
```

**Fix** (replace the sprint number with the correct one for your branch):
```bash
for f in packages/sap.fe.{core,macros,navigation,plugins.managecache,templates,test,tools}/ui5.yaml; do
  sed -i '' 's/version: true #FrameworkVersion#/version: 1.120.51-SNAPSHOT #FrameworkVersion#/' "$f"
done
```
Commit message: `[INTERNAL] UI5 Versionbump Fix`  
Push to `refs/for/rel-1.X`. Wait for V+1 before proceeding.

---

### 2. SNAPSHOT artifact in `build-releases` Artifactory

**Symptom**: xmake Release job fails with:
```
Could not resolve dependencies ... com.sap.ui5:sapui5-dist-library-parent:pom:1.120.51-SNAPSHOT
```
**Cause**: A `-SNAPSHOT` version is still in one of the Maven POMs. The release Artifactory repo only accepts stable versions.  
**Fix**: Create the Manual Pre-Release Commit (see Recovery Workflow above).

---

### 3. Wrong release version after premature post-release

**Symptom**: Branch is at `1.X.44-SNAPSHOT` but `1.X.43` was never deployed.  
**Cause**: Post-release automation runs even when the build fails — so the version is already bumped.  
**Fix**: Your target release version is `1.X.43` (current SNAPSHOT minus 1). Use that in the Manual Pre-Release Commit. Confirm with the DevOps architect first.

---

### 4. Gerrit Change-Id missing

**Symptom**: `git push` fails with `missing Change-Id in message footer`  
**Cause**: Worktrees don't inherit the Gerrit commit-msg hook automatically.  
**Fix**: Generate the Change-Id manually (see Step E in Recovery Workflow above).

---

### 5. Approvals reset after amend

After `git commit --amend + push`, all CR+2 and V+1 votes are **reset** on Gerrit — even if nothing significant changed. This is normal Gerrit behavior. Ask the approvers to re-review the new patchset.

---

### 6. Post-release nightly build fails: `package.json version does not match pom.xml`

**Symptom**: Nightly build on the rel branch fails after the release with:
```
package.json: version "1.X.43" does not match version defined in pom.xml: "1.X.44-SNAPSHOT"
```

**Cause**: The automated post-release is split across two bots — neither bumps the npm `package.json` files:
- `pxmaktagger` (xmake): bumps `maven/sap.fe-collection/pom.xml` + root `pom.xml`
- `sapui5` bot: bumps `maven/sap-fe-parent/pom.xml` + `maven/open.fe/pom.xml`

The `npm-validator` check requires `package.json` and `pom.xml` versions to match — so the build fails.

**This is not your fault** — known automation gap. Report to the DevOps architect.

**Fix** (replace `1.X.43` and `1.X.44` with actual version numbers):
```bash
# 1. Create a worktree from the rel branch
git worktree add .claude/worktrees/fix-rel-1.X origin/rel-1.X
cd .claude/worktrees/fix-rel-1.X

# 2. Find all files still at the old released version
grep -rl '1\.X\.43' --include="*.json" . | grep -v node_modules | grep -v .git | grep -v target

# 3. Bump all of them to the new SNAPSHOT version
grep -rl '1\.X\.43' --include="*.json" . | grep -v node_modules | grep -v .git | grep -v target \
  | xargs sed -i '' 's/1\.X\.43/1.X.44-SNAPSHOT/g'

# 4. Verify nothing remains at the old version
grep -r '1\.X\.43' --include="*.json" . | grep -v node_modules | grep -v .git

# 5. Stage, commit, push
git add -A
git commit -m "[INTERNAL][VERSION CHANGE ONLY] Bump package.json versions to 1.X.44-SNAPSHOT"
# If push rejects with "missing Change-Id", add one:
CHANGE_ID="I$(git rev-parse HEAD | openssl dgst -sha1 | awk '{print $2}')"
git commit --amend -m "$(git log -1 --format='%B')
Change-Id: $CHANGE_ID"
git push origin HEAD:refs/for/rel-1.X
```

Files typically affected (~20 files): `packages/*/package.json`, `maven/*/package.json`, `lib/docbuilder/package.json`, `lib/docbuilder/src/sap-ui-version.json`.

> ⚠️ Do **not** trigger the Release job after this fix. This only restores the branch to the correct post-release development state.

---

## Post-Branching Tasks (when a new rel-1.X branch is created)

When the UI5 Infrastructure team creates a new release branch (e.g. `rel-1.153`), one manual commit is needed on **master**. This is separate from the release process — it happens once per sprint when the branch is cut.

Contact for branch creation: DL_541983F5DF15DB0B44000004@global.corp.sap

**Commit message**: `[INTERNAL] post branching bump`  
**Target**: push to `refs/for/master` (NOT the rel branch)

**Wait for Renovate first**: Renovate manages updates for Maven dependencies, YAML files, and `@ui5/cli` automatically and runs daily. In most cases it will update dependencies on the day after the branch is cut. Only proceed manually if Renovate hasn't run yet or for the special cases below.

> **Note — NPM dependencies are NOT covered by Renovate** and must be updated manually if needed.

**4 files to change** (example: when `rel-1.153` is created):

**1. `.github/workflows/Synopsys_BD_Scans.yml`** — add the new branch to the BlackDuck scan list.  
Rule: only ADD, never REMOVE existing branches unless the architect explicitly confirms.
```yaml
# Add new branch at the front, keep all others:
run: echo 'branches=["main", "rel-1.153", "rel-1.152", "rel-1.151", "rel-1.148", "rel-1.136"]' >> $GITHUB_OUTPUT
```

**2. `packages/sap.fe.controls/ui5-local.yaml`** — bump the UI5 local version to the new sprint number.

**3. `packages/sap.fe.placeholder/ui5-local.yaml`** — bump the UI5 local version to the new sprint number.

**4. `utils/pom.xml`** — bump all SAP dependency versions to the new sprint:
```xml
<!-- All of these become 1.{new-branch-number}.0-SNAPSHOT -->
<!-- e.g. when rel-1.153 is created: 1.152.0-SNAPSHOT → 1.153.0-SNAPSHOT for: -->
com.sap.ushell.version, com.sap.cards.version, com.sap.ovp.version,
com.sap.genTemplate.version, com.sap.zen, com.sap.feedBackUi.version
```
The version number always matches the new branch number — no lookup needed.

> **Exception — two external dependencies** (`nw.core.gbt.notes.lib.reuse.version` and `ux.eng.fioriai.reuse.version`): these are tagged `<!--DependencyVersion,manualChange-->` and their SNAPSHOT artifacts can vanish from Artifactory.
>
> 1. Check latest version at the links in Quick Reference above.
> 2. Verify the artifact exists in Artifactory before committing.
> 3. **Fallback if SNAPSHOT is not available**: use the latest stable released version closest to the SNAPSHOT (e.g. `2611.0.0` instead of `2611.0.0-SNAPSHOT`). Renovate should eventually pick this up, but it is currently not working for these two dependencies.
>
> Both properties track the same sprint version — bump them together.

Reference commit: Gerrit 8033826 (Frederic, Aug 19 2026 — added rel-1.152 + rel-1.153).

---

## Reference Commits on rel-1.120

| Commit | What it did |
|---|---|
| `baf1ad369a` | Pre-release for 1.120.43 (FAILED — had YAML bug AND wrong snapshot deps) |
| `041022ca51` | Premature post-release bump to 1.120.44-SNAPSHOT (ran despite failed build) |
| Gerrit 8127215 | Corrected manual pre-release commit for 1.120.43 |
| Gerrit 8134586 | Manual package.json version bump fix (Known Failure Mode #6) |

Use `git show <sha>` to inspect the exact diff pattern for any of these.
