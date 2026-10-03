# GitHub 自动化模板（PostRelease / Promote Prerelease）

两个 GitHub Actions 工作流的**通用模板**与要点，可直接套用到任意仓库。

`<项目名>` 为占位符，套用时整体替换为目标仓库名；分支名不写死，取仓库默认分支，因此主分支叫 `master`、`main` 还是别的名字都无需修改。

本模板**按需取用**：只有确实需要该发布自动化的仓库才引入，不必每个仓库都放。

## 概览

| 文件 | 触发方式 | 作用 |
| --- | --- | --- |
| `post_release.yml` | 发布 Release（`published`） | 校验并上传发布包 zip，把 `.releasenote.md` 作为发布说明，然后从主分支清除 `output/` |
| `promote_prerelease.yml` | 每日 22:00 UTC 定时 + 手动触发 | 把发布满 48 小时的最新 Pre-Release 提升为 Latest |

两个工作流都声明 `permissions: contents: write`，使用仓库自带的 `GITHUB_TOKEN`，**无需配置任何自定义 Secret**。

## 发布流程全貌（重点）

```text
本地打包 → zip 提交进 output/ → 打 vX.Y.Z tag 并发布 Release
                                      │
                                      ▼
        post_release.yml 触发：校验 zip → 上传为 Release 资产
                              正文取自 .releasenote.md
                              → 再把 output/ 从主分支删除并推送
                                      │
                         （发布时勾选 Pre-Release 的话）
                                      ▼
        promote_prerelease.yml：48 小时后自动转为 Latest
```

要点小结：

1. zip 必须**先提交到主分支的 `output/`**，路径与文件名固定为 `output/<项目名>_<版本>.zip`。
2. tag 必须以 `v` 开头（如 `v1.2.3`），版本号取自 tag 去掉 `v`。
3. Release 正文不写在 GitHub 界面上，而是由仓库根目录的 `.releasenote.md` 提供——与仓库守则第 5 条直接挂钩。
4. 发布成功后 `output/` 会被自动删除并提交，二进制不留在仓库里。
5. 预发布需要 48 小时冷静期才转正，期间可用 `[no-release]` 标记阻止转正。

---

## 工作流一：`post_release.yml`

`<项目名>` 为占位符，套用时整体替换为目标仓库名。

```yaml
name: <项目名>-PostRelease
on:
  release:
    types: [published]

permissions:
  contents: write

jobs:
  upload_asset:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Extract base version from tag
        id: get_version
        run: |
          if [[ "${{ github.event.release.tag_name }}" != v* ]]; then
            echo "Skipping because tag does not start with 'v'"
            exit 0
          fi
          BASE_VERSION=${{ github.event.release.tag_name }}
          BASE_VERSION=${BASE_VERSION#v}
          echo "VERSION=$BASE_VERSION" >> $GITHUB_OUTPUT
          echo "Expected version is $BASE_VERSION"

      - name: Check Zip File Existence
        if: steps.get_version.outputs.VERSION != ''
        run: |
          ZIP_PATH="output/<项目名>_${{ steps.get_version.outputs.VERSION }}.zip"
          if [ ! -f "$ZIP_PATH" ]; then
            echo "❌ Error: Could not find zip file at $ZIP_PATH"
            exit 1
          else
            echo "✅ Found zip file: $ZIP_PATH"
          fi

      - name: Upload Asset to Release
        if: steps.get_version.outputs.VERSION != ''
        uses: softprops/action-gh-release@v1
        with:
          body_path: .releasenote.md
          files: output/<项目名>_${{ steps.get_version.outputs.VERSION }}.zip

      - name: Cleanup output directory
        if: steps.get_version.outputs.VERSION != ''
        run: |
          git config user.name "eric-bot"
          git config user.email "ericzhang233@outlook.com"
          BRANCH="${{ github.event.repository.default_branch }}"
          git fetch origin "$BRANCH"
          git checkout "$BRANCH"
          if [ -d "output" ]; then
            git rm -r output/
            git commit -m "chore: auto-remove output directory after release"
            git pull --rebase origin "$BRANCH"
            git push origin "$BRANCH"
          fi
```

### 逐步说明

| 步骤 | 行为 |
| --- | --- |
| Checkout | 检出仓库（当前工作区是触发时的 tag） |
| Extract base version | 校验 tag 前缀；去 `v` 得版本号，写入 `$GITHUB_OUTPUT` 的 `VERSION` |
| Check Zip File Existence | 断言 `output/<项目名>_<版本>.zip` 存在，否则 `exit 1` 失败 |
| Upload Asset | 用 `softprops/action-gh-release@v1` 上传 zip，正文取 `.releasenote.md` |
| Cleanup output | 切到主分支，删除并提交 `output/`，推送 |

### 重点与注意

- **非 `v*` tag 会静默跳过**：第一个步骤执行 `exit 0`（步骤本身算成功），`VERSION` 为空，后续三个步骤被 `if: steps.get_version.outputs.VERSION != ''` 全部跳过。因此 tag 写成 `1.2.3` 而不是 `v1.2.3` 时，**工作流显示绿色但什么都没做**，Release 里不会有任何资产。
- **文件名是硬约定**：`output/<项目名>_<版本>.zip` 三段必须完全吻合，`<项目名>` 替换为目标仓库名。
- **发布说明来自 `.releasenote.md`**：`body_path` 指向仓库根目录的文件，所以发版前必须先按母法 `releasenoteguide.md` 的规范写好 `.releasenote.md`，否则发布出去的是上一次的旧内容或空正文。
- **清理步骤固定在主分支上操作**：`git fetch` / `git checkout` / `git pull --rebase` / `git push` 四处都作用于仓库的**主分支**，取值来自 `github.event.repository.default_branch`，因此无需按仓库修改分支名。
- **工作区切到主分支是刻意的**：因为要删的是主分支上的 `output/`，而不是触发发布的那个 tag。
- **提交身份写死**：`eric-bot` / `ericzhang233@outlook.com`，用于 `chore: auto-remove output directory after release` 这一次提交。
- **推送权限**：依赖 `permissions: contents: write`；若主分支开启了分支保护或禁止机器人推送，`git push` 会被拒绝，此时 zip 已上传成功但 `output/` 不会被清理。
- **`output/` 不存在就跳过**：`if [ -d "output" ]` 判断以目录存在为前提。若将来改为 CI 直接构建 zip（不再提交 `output/`），此步骤会静默跳过，属于预期行为。
- **没有并发控制**：工作流未声明 `concurrency`。连续快速发布两个 Release 时，两个 job 可能同时推主分支；`git pull --rebase` 只能缓解、不能完全避免冲突。
- **必须先提交 zip 再发布**：`Check Zip File Existence` 从当前 checkout 的工作区读文件，zip 不在主分支上就会直接失败。

---

## 工作流二：`promote_prerelease.yml`

```yaml
name: Promote Prerelease to Latest

on:
  schedule:
    - cron: '0 22 * * *'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
      - name: Promote Prereleases older than 48 hours
        uses: actions/github-script@v7
        with:
          script: |
            const owner = context.repo.owner;
            const repo = context.repo.repo;
            const now = new Date();

            const { data: releases } = await github.rest.repos.listReleases({ owner, repo, per_page: 20 });

            if (!releases.length) {
              console.log("No releases found.");
              return;
            }

            const latestRelease = releases[0];
            if (!latestRelease.prerelease) {
              console.log(`Latest release ${latestRelease.tag_name} is already latest. Skipping.`);
              return;
            }

            const latestPre = latestRelease;

            if (latestPre && latestPre.published_at) {
              if (latestPre.body && latestPre.body.toLowerCase().includes('[no-release]')) {
                console.log(`Skipping latest prerelease ${latestPre.tag_name} due to [no-release] tag.`);
              } else {
                const publishedAt = new Date(latestPre.published_at);
                const hoursSincePublish = (now - publishedAt) / (1000 * 60 * 60);

                if (hoursSincePublish >= 48) {
                  console.log(`Promoting release ${latestPre.tag_name} to latest`);
                  await github.rest.repos.updateRelease({
                    owner,
                    repo,
                    release_id: latestPre.id,
                    prerelease: false,
                    make_latest: "true"
                  });
                } else {
                  console.log(`Latest prerelease ${latestPre.tag_name} has not reached 48 hours yet (${hoursSincePublish.toFixed(1)}h).`);
                }
              }
            } else {
              console.log("No prerelease found.");
            }
```

### 判定流程

```text
取最新 20 个 Release 的第 1 个（= 最新的那个）
  ├─ 没有 Release            → 结束
  ├─ 不是 Pre-Release        → 结束（不回溯更早的预发布）
  ├─ 正文含 [no-release]     → 跳过本次提升
  ├─ 发布未满 48 小时        → 本次不提升，等下次定时
  └─ 发布已满 48 小时        → 设为正式版 + 标记 Latest
```

### 重点与注意

- **触发时间**：`cron: '0 22 * * *'` 使用 UTC，对应**北京时间次日 06:00**。GitHub 定时任务在仓库长期无活动时可能被自动停用，需留意 Actions 页面的提示。
- **只看最新的那一个**：代码取 `releases[0]`（GitHub 按创建时间倒序，最新在前）。若最新 Release 不是预发布，直接 `return`，**不会去扫描更早的预发布**。也就是说一天发多个预发布时，只有最新的那个会被提升，旧的预发布会一直保留。
- **48 小时冷静期**：以 `published_at` 计算（不是 tag 创建时间），满 48 小时才执行 `updateRelease`。
- **`[no-release]` 开关**：写在**Release 正文**中（大小写不敏感）。用于"这个预发布只做测试、不要转正"的场景。注意正文由 `post_release.yml` 从 `.releasenote.md` 填充，所以这个标记必须在发版前写进 `.releasenote.md`，之后在 GitHub 上手改正文也行。
- **幂等且自终止**：提升后该 Release 不再是预发布，下次运行时第一步判断就 `return`，因此不需要任何额外状态记录，重复运行是安全的。
- **固定取 20 条、无分页**：`per_page: 20` 是写死的。正常情况下最新一条就足够，只有在极短时间内堆出大量 Release 时才可能受影响。
- **可手动补跑**：`workflow_dispatch` 提供按钮式手动触发，适合发版后想立刻转正的场景。
- **不涉及分支与提交**：整个过程只调用 Releases API，不写代码、不推提交，因此与 `permissions: contents: write` 之外的权限无关，也不会受分支保护影响。

---

## 按需取用到某个仓库的检查清单

1. 只把需要的工作流放到该仓库 `.github/workflows/`，不需要的不要放。
2. 把 `post_release.yml` 里的 `<项目名>` 替换为目标仓库名：`name` 一处，zip 路径两处（`Check Zip File Existence` 与 `Upload Asset`）。
3. 分支名无需处理：模板取 `github.event.repository.default_branch`；若要固定成某个分支名，改 `Cleanup output directory` 里的 `BRANCH=` 一行即可。
4. 在仓库 Settings → Actions → General 中允许工作流读写仓库内容（对应 `contents: write`），否则清理步骤无法推送。
5. 确保仓库根目录存在 `.releasenote.md`，且发版前已按母法 `releasenoteguide.md` 更新。
6. `promote_prerelease.yml` 无需修改即可使用。

## 与仓库守则的关系

- 通用守则**第 5 条 Release Note 规则**：本自动化把 `.releasenote.md` 直接当作 Release 正文，所以"发版前先按母法 `releasenoteguide.md` 写好 `.releasenote.md`"不是可选项，而是该流程能否产出正确发布说明的前提。
- 通用守则**第 9 条 提交信息署名原则**：清理步骤产生的提交为 `chore: auto-remove output directory after release`，不含任何工具或助手署名，符合该条要求。
