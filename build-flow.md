# 编译流规范

按需引入的编译、打包与清理规范：不是所有仓库都要遵守，新项目按需取用。

全员无条件适用的规则在母法 [agent.md](./agent.md)。

---

## 目标

采用本流程的仓库，在 **Release 构建**时自动产出与发布工作流对接的发布包：

```text
output/<项目名>_<版本>.zip
```

这个路径与命名是硬约定——`.github/workflows/post_release.yml` 正是按它查找并上传的。

## 四条不可变的约定

1. **版本号只有一个来源**：仓库根目录的一个版本文件是唯一真值，构建期自动解析，其它任何地方不得重复写版本号。
   - .NET 项目：`Version.cs` 里 `Version = "X.Y.Z"`
   - CMake / C++ 项目：`Version.h` 里 `VERSION_STRING "X.Y.Z"` 与产品名
2. **产物固定为 `output/<项目名>_<版本>.zip`**，其中 `<版本>` 与版本文件一致，**不带 `v` 前缀**。
3. **只在 Release 配置打包**：Debug 构建不产出发布包，避免误把调试产物发出去。
4. **中间产物不在仓库留存**：构建与 `clean` 都要物理删除 `bin`、`obj`、暂存目录；唯一保留的是 `output/` 里的发布包（它是发布流程的输入）。

## 参考实现一：EricGameLauncher（.NET / MSBuild）

- **版本源**：`Version.cs`
- **版本解析**：`.csproj` 在求值阶段读取 `Version.cs` 并用正则提取，得到 `$(ExtractedVersion)`；再由 `SyncVersion` 目标写入 `AssemblyVersion` / `FileVersion`。
- **打包目标**：构建后触发，仅 Release 生效。

```xml
<Target Name="ZipReleaseBuild" AfterTargets="Build" Condition="'$(Configuration)' == 'Release'">
  <MakeDir Directories="output" />
  <ZipDirectory SourceDirectory="$(OutputPath)" DestinationFile="output\EricGameLauncher_$(ExtractedVersion).zip" Overwrite="true" />
</Target>
```

替换 `<项目名>` 即可复用到其它 .NET 仓库。

- **配套约定**：
  - 子组件由主工程统一驱动，不需要分别手工构建：CLI 项目构建后复制进输出目录；影子更新组件走 payload 机制嵌入主程序（见「payload 机制」）。
  - 打包之后与 `clean` 之时都要清理中间产物，见下面的「clean 操作」。

## 参考实现二：VolumeMixerExtender（CMake）

- **版本源**：`Version.h`
- **版本解析**：`CMakeLists.txt` 正则解析出版本与产品名，生成版本资源脚本，并定义：
  - `<PACKAGE_NAME> = <PRODUCT_NAME>_<VERSION>`
  - `<PACKAGE_DIR> = <源目录>/output`
- **打包脚本**：`cmake/Package.cmake`，逻辑三步：
  1. **非 Release 直接 `return()`**，不产出发布包；
  2. 把可执行文件暂存到 `output/<PACKAGE_NAME>/`，用 `cmake -E tar cf <PACKAGE_NAME>.zip --format=zip` 打包；
  3. 删除暂存目录，只留 zip。
- **触发方式**：既挂 `add_custom_command(TARGET <主目标> POST_BUILD ...)` 自动执行，也提供显式 `package` 目标供手动重打。

## payload 机制：构建期嵌入内部组件

需要随程序分发、但不该出现在发布目录里的内部组件（更新器、注入器等），统一在**构建期嵌入主程序**，运行时再释放到临时目录执行。发布目录因此保持干净，组件也无法被外部替换或篡改。

两种实现路线：

| 路线 | 做法 | 清理负担 |
| --- | --- | --- |
| 暂存目录 + 嵌入资源（EGL） | 先把子项目发布到 `payload/<组件>/`，再把该目录收进主程序集的 `EmbeddedResource` | 有：嵌入完成后必须删除 `payload/` |
| 直接引用子目标产物（VMEX） | 生成资源脚本，用 `RCDATA` 直接引用子可执行文件的构建产物路径 | 无：不存在暂存目录 |

### EGL：payload + EmbeddedResource

三步，顺序不可颠倒：

1. `BuildMainUpdater` / `BuildCfgUpdater`（`BeforeTargets="BeforeBuild"`）：用 MSBuild 调用子项目的 `Restore;Publish`，产物落到 `payload/updater.main/`、`payload/updater.cfgver/`。
2. `EmbedPayloadResources`（`BeforeTargets="AssignTargetPaths"`）：把 payload 收进 `EmbeddedResource`，并指定 `LogicalName`。
3. `PostBuildCleanupPayload`（`AfterTargets="Build"`）：删除 `payload/`。

```xml
<Target Name="EmbedPayloadResources" BeforeTargets="AssignTargetPaths">
  <ItemGroup>
    <_PayloadMain Include="payload\updater.main\*.*" />
    <_PayloadCfg Include="payload\updater.cfgver\*.*" />
  </ItemGroup>
  <ItemGroup>
    <EmbeddedResource Include="@(_PayloadMain)" LogicalName="EricGameLauncher.updater.main.%(Filename)%(Extension)" />
    <EmbeddedResource Include="@(_PayloadCfg)" LogicalName="EricGameLauncher.updater.cfgver.%(Filename)%(Extension)" />
  </ItemGroup>
</Target>
```

**顺序是硬要求**：`BeforeBuild`（第 1 步）必须早于 `AssignTargetPaths`（第 2 步）。反过来的话嵌入时 payload 还不存在，资源会**静默缺失**——构建成功，但程序运行时解不出组件。

### VMEX：RCDATA 直接嵌入

`cmake/GeneratePayloadRc.cmake` 生成资源脚本，把子可执行文件按**构建产物路径**写进 `RCDATA`：

```rc
VMEX_PAYLOAD_ID_LAUNCHER RCDATA "<launcher 产物路径>"
VMEX_PAYLOAD_ID_TAP RCDATA "<tap 产物路径>"
```

`CMakeLists.txt` 用 `add_custom_command(OUTPUT ... DEPENDS $<TARGET_FILE:vmex_launcher> $<TARGET_FILE:vmex_tap>)` 保证先生成子目标，再把该 `.rc` 作为源文件编进主程序。不产生暂存目录，因此也不需要对应的清理。

### 通用要求

- 内部组件**不散落在发布目录**，分发由构建期嵌入承担。
- 嵌入步骤必须早于资源收集阶段，且对子目标声明明确依赖。
- 若走了暂存目录路线，暂存目录用完即删，不留仓库。

## clean 操作

打包流的另一半是清理：构建过程不允许把中间产物留在仓库里。三个时机，互不替代：

| 时机 | 目标（EGL） | 目的 |
| --- | --- | --- |
| 每次构建后（任何配置） | `PostBuildCleanupPayload` | 擦掉 payload 暂存区——组件此刻已嵌入主程序集，该目录已是垃圾 |
| Release 打包后 | `ReleasePostCleanPayload` | 让仓库回到「只有源码 + `output/` 里的 zip」 |
| 执行 `clean` 时 | `CleanAllPayload` | 在标准 clean 之外再物理删除，避免 `dotnet clean` 留下残渣 |

删哪些目录由工程自己维护，本规范不逐一罗列——**新增子组件时记得把它补进清单**，否则它的 `bin`/`obj` 会一直留在仓库里。

```xml
<Target Name="PostBuildCleanupPayload" AfterTargets="Build">
  <RemoveDir Directories="payload" />
</Target>
```

`ReleasePostCleanPayload` 与 `CleanAllPayload` 形式与此相同，只是清单更长（主工程与全部子组件的 `bin`/`obj`，再加 `payload`）；两者清单一致，区别只在触发时机（打包之后 vs `Clean` 之后）。

要点：

- 三个目标一律用 `RemoveDir` **物理删除**，不再依赖各自的标准清理机制，确保任何残渣都被抹掉。
- `PostBuildCleanupPayload` 不设配置条件，Debug 与 Release 都执行；后两个只在 Release 生效（只有 Release 会打包）。
- **`output/` 故意不被忽略**：它是发布流程的输入（需提交到主分支供 `post_release.yml` 上传），所以 `.gitignore` 不忽略它；但要把它排除在源码扫描之外（EGL 用 `<DefaultItemExcludes>` 加入 `output\**`）。

CMake 侧的对应做法：

- VMEX 没有自定义 clean 目标，依赖 CMake 标准 clean，`bin/`、`obj/`、`build/` 交由 `.gitignore` 忽略。
- 打包脚本自己收尾：`Package.cmake` 在 zip 生成后 `file(REMOVE_RECURSE ...)` 删除暂存目录 `output/<PACKAGE_NAME>/`，仓库里只留 `output/` 下的 zip。

## 与发布工作流衔接

采用本流程的仓库，发版顺序为：

1. 更新版本文件里的版本号。
2. 按母法 `releasenoteguide.md` 的规范更新 `.releasenote.md`。
3. 执行 Release 构建，得到 `output/<项目名>_<版本>.zip`。
4. 把该 zip 提交到主分支。
5. 打 `v<版本>` tag 并发布 Release；若该仓库取用了 `post_release.yml`，它会自动把 zip 上传为 Release 资产，并把 `output/` 从主分支清除。

**关键点**：tag 中的版本号必须与 zip 文件名中的版本号完全一致，否则工作流找不到文件而失败。

## 通用检查清单

- [ ] 版本号只有一个来源文件，构建期自动解析，无重复书写
- [ ] Release 构建产出 `output/<项目名>_<版本>.zip`
- [ ] Debug 构建不产出发布包
- [ ] 打包逻辑写在工程文件内（`.csproj` / `CMakeLists.txt`），不依赖手工压缩
- [ ] 需随程序分发的内部组件在构建期嵌入主程序，不散落在发布目录
- [ ] 嵌入步骤早于资源收集阶段，且对子目标声明依赖
- [ ] 构建后与 `clean` 时都物理删除中间目录（`bin`、`obj`、`payload`、打包暂存目录等）
- [ ] `output/` 不被 `.gitignore` 忽略（发布流程的输入），但被排除在源码扫描外
- [ ] 若接入 `post_release.yml`：zip 的 `<项目名>` 前缀与工作流中的一致
