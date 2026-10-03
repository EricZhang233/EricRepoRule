# EricRepoRule

EricZhang 所有仓库的规则中心与唯一权威来源。

本仓库整体是**母法**，全组织只有这一份，**永远不复制到其它仓库**；各仓库根目录的 `agent.md` 用首行指针指向本仓库，指针之后才是该仓库的私有规范。

## 架构：母法 + 私有规范

```text
EricRepoRule/               ← 母法（整个仓库），唯一一份，所有仓库共享
        ▲
        │ 首行指针
<任意仓库>/agent.md         ← 第 1 行指向本仓库，其余为本仓库私有规范
```

- **母法**：本仓库整体，包含 [agent.md](./agent.md)（通用永久守则第 1–9 条）、[releasenoteguide.md](./releasenoteguide.md)（发布说明规范）、[build-flow.md](./build-flow.md)（按需引入的编译流规范）、[github-workflows.md](./github-workflows.md)（工作流模板与要点）等。修订只在本仓库进行，改完全部仓库立即生效，无需同步。
- **按需规范**：[build-flow.md](./build-flow.md) 收录的规范是**按需引入**的，仓库需要时才遵守，不强制全员。
- **私有规范**：各仓库根目录 `agent.md` 中指针之后的区域，只写该仓库特有的补充。
- 各仓库中私有的规范均需以本仓库所有规范为母法，然后再在各仓库私有规则文件下写私有规范。

目标仓库 `agent.md` 的标准形态——第一行**永远**是指向本仓库的指针（指向仓库根即可，不必指到单个文件）：

```markdown
母法：https://github.com/EricZhang233/EricRepoRule —— 需要完全读取并遵守此仓库所有规范，本文件其余部分为本仓库私有规范。

---

## 本仓库私有规范

- ......
```

## 文件一览

本仓库的规定内容**只存在于本仓库**，其它仓库一律不复制规则文本，只放一个指向本仓库的指针。

| 文件 | 用途 | 是否进入其它仓库 |
| --- | --- | --- |
| [agent.md](./agent.md) | 通用永久守则（第 1–9 条），母法主体 | 否，仅被各仓库 `agent.md` 的首行指针引用 |
| [releasenoteguide.md](./releasenoteguide.md) | 发布说明编写规范，配合守则第 5 条 | 否，规范正文只在本仓库 |
| [build-flow.md](./build-flow.md) | **按需引入**的编译流规范（编译 / 打包 / 清理） | 否（规则层面）；仓库按需遵守，规则正文只在本仓库 |
| [devrules.md](./devrules.md) | 个人开发习惯（配置与缓存的存放位置等），参考性内容 | 否，仅存在于本仓库 |
| [github-workflows.md](./github-workflows.md) | `post_release.yml` 与 `promote_prerelease.yml` 的通配模板与要点说明 | 否（规则层面）；**按需**取用——只有确实需要该发布自动化的仓库才把 YAML 落到自己的 `.github/workflows/`（Actions 要求工作流文件必须在仓库内） |
| README.md | 本文件：用法与维护说明 | 否，仅存在于本仓库 |

其它仓库里会出现的东西只有两类：**指向本仓库的指针**，以及**按需取用的工作流文件**。规则文本一概不复制，工作流也不无脑下发。

## 套用步骤：挂载母法

1. **挂载母法**：在目标仓库根目录建 `agent.md`，第一行照抄上一节的标准指针，其后写该仓库的私有规范。**不要复制母法条文**。
2. **发布说明本体**：在目标仓库根目录建 `.releasenote.md`，撰写时遵循母法 [releasenoteguide.md](./releasenoteguide.md) 的规范（该规范不复制到目标仓库）。
3. **可选**：目标仓库若要使用 Copilot 的仓库指令机制，可建 `.github/copilot-instructions.md`：

   ```text
   # 永久守则
   本仓库以 EricRepoRule 为母法：https://github.com/EricZhang233/EricRepoRule
   遵循母法中的永久守则，并遵守仓库根目录 agent.md 中的私有规范。
   ```

## 按需取用：发布工作流

`github-workflows.md` 里的两个工作流**不是每个仓库都要**，只在仓库确实需要对应的发布自动化时才引入。

1. **只取需要的那一个**：把工作流放到目标仓库 `.github/workflows/` 下。
   - `promote_prerelease.yml`：预发布到期自动转正，与项目名无关，可直接使用。
   - `post_release.yml`：发布后上传资产并清理 `output/`，需把 `<项目名>` 替换为目标仓库名（`name` 一处、zip 路径两处）。
   - 分支名无需处理：模板取仓库默认分支，主分支叫 `master` 还是 `main` 都不用改。
2. **仓库权限**：引入了工作流之后再开 Settings → Actions → General 允许读写仓库内容（对应 `contents: write`），否则清理步骤无法推送。

## 按需引入：编译流

[build-flow.md](./build-flow.md) 里的编译流规范同样是按需引入：仓库想让 Release 构建自动产出 `output/<项目名>_<版本>.zip`、并在打包与 `clean` 后不留中间产物，才照它改造自己的工程文件；不采用该流程的仓库不受影响。

## 维护约定

- 母法只有一份：规则改动都写在本仓库（[agent.md](./agent.md) 等），所有仓库立即生效，不要复制到其它仓库。
- 各仓库 `agent.md` 的**第一行永远指向本仓库**，不得改成仓库内的相对路径或本地副本。
- 私有规范只能补充母法，不得与母法冲突；冲突时以母法为准。
- 规则文本一律不复制到其它仓库：母法内容只存在于本仓库，目标仓库只保留指向本仓库的指针。
- 母法里不写只在某个仓库成立的内容（具体仓库名、该仓库的目录结构等）。
