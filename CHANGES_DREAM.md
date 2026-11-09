# Dream 定制版改动说明

基于 Cloudreve 上游标签 `4.17.0`（`d4a145c`），主仓库和 `assets` 前端子模块均使用 `dream-4.17.0` 分支。构建步骤和启动排查见 [构建说明](BUILD_DREAM.md)。

## 使用方式

适用于单用户、本地存储，并且不同文件不会占用同一个物理路径。在管理后台编辑正在使用的本地存储策略：

- **目录规则**：保留新策略预填的 `data/cloudreve/{path}` 对应实际路径（可能显示为绝对路径）。如果想用不带 `data/` 的相对路径，可以明确填写 `cloudreve/{path}`。
- **Blob 名称**：填写 `{originname}`，保留上传时的原文件名，不附加 `{uid}` 或随机字符串。
- `{path}` 是上传目标在 Cloudreve 中的目录，不是浏览器所在电脑的路径。例如将 `笔记.txt` 上传至 `资料/2026`，使用 `cloudreve/{path}` 时，新 Blob 的相对路径是 `cloudreve/资料/2026/笔记.txt`。

**已有策略不会自动修改。** 首次初始化的默认本地策略仍以 `{uid}_{randomkey8}_{originname}` 作为 Blob 名称，新建本地策略的向导仍预填 `{uuid}_{originname}`；都需要在后台手动改成 `{originname}`。

## 代码改动

| 位置 | 相对上游的改动 |
| --- | --- |
| [`StorageAndUploadSection.tsx`](assets/src/component/Admin/StoragePolicy/EditStoragePolicy/FormSections/StorageAndUploadSection.tsx) | 移除表单对 `{uuid}`、`{randomkey8}`、`{randomkey16}` 至少出现一次的限制。保留必填校验和可选变量。前端提交 `a405542`。 |
| [`inventory/migration.go`](inventory/migration.go)、[`service/admin/policy.go`](service/admin/policy.go) | 首次插入默认本地策略、后台创建本地策略时，目录改用 `util.DataPath("cloudreve/{path}")`，移除目录中的 `{uid}`。已有默认策略会跳过初始化；后台编辑策略本来就不要求随机变量。 |
| [`application/migrator/policy.go`](application/migrator/policy.go) | 旧版 v3 策略迁移时，不再因缺少随机变量而覆盖规则。目录或名称各自为空时，分别补 `cloudreve/{path}` 和 `{originname}`；已有非空规则保留。 |
| [`.gitmodules`](.gitmodules)、[`assets/package.json`](assets/package.json) | 子模块指向前端 fork。前端提交 `118ce06` 将资源版本从 `4.0.0-next` 调整为 `4.17.0-dream`，用于匹配后端构建版本。 |

## Blob 行为边界

规则仍由原有的 [`generateSavePath`](pkg/filemanager/fs/dbfs/dbfs.go) 展开；驱动、Blob 引用和读写逻辑未随这次定制修改。改变规则不会重命名、搬迁已经写入的 Blob；网盘里的移动或改名也不应视为磁盘目录同步。不要绕过 Cloudreve 直接移动或改名磁盘中的已登记文件。

原名规则会复用相同的物理路径。本地驱动对普通非覆盖上传仍检查目标是否已存在；但回收站保留旧 Blob、历史版本或大小写不敏感的文件系统，都可能使界面上看似不同的文件占用同一路径。不要把该规则用于共享同一存储目录的多个用户。对象存储和远程存储向导的默认规则未改。

## 核对结果与发布检查

- 主仓库原先记录的前端提交 `a405542` 只包含 Blob 表单修改；现在将 `assets` 指针同步到 `118ce06`，包含前端版本修正。前端 fork 的 `dream-4.17.0` 已包含该提交。
- 已检查本地 `assets/build/version.json` 和 `application/statics/assets.zip` 中的版本均为 `4.17.0-dream`；此检查不代表部署服务器当前运行的二进制或外部静态资源也已更新。
- 升级后建议用临时目录验证：保存无随机变量的策略、上传和下载、不同目录同名文件、同目录冲突、删除进回收站后再次上传。涉及覆盖或历史版本时，先用测试文件确认行为，再用于正式数据。
