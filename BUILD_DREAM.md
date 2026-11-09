# Dream 定制版构建说明

存储策略改动、后台设置和 Blob 使用边界见 [Dream 定制改动说明](CHANGES_DREAM.md)。

本项目包含一个前端子模块：

```text
cloudreve 主仓库  -> git@github.com:dreamstation625/cloudreve.git
assets 前端仓库   -> git@github.com:dreamstation625/cloudreve-frontend.git
```

前后端都使用同名分支：

```text
dream-4.17.0
```

## 版本号规则

前端静态资源包 `application/statics/assets.zip` 内 `assets/build/version.json` 的 `version` 应与后端二进制里的 `BackendVersion` 一致。Vite 从 `assets/package.json` 的版本生成资源版本；只改 PowerShell 的 `$env:VERSION` 不会自动更新已生成的资源包。

如果不一致，Cloudreve 启动时可能提示静态资源版本不匹配。

推荐定制版本号：

```powershell
$env:VERSION = "4.17.0-dream"
```

如果前端用 `4.17.0` 构建，后端也必须用 `4.17.0` 构建。

如果前端用 `4.17.0-dream` 构建，后端也必须用 `4.17.0-dream` 构建。

## Windows 下构建前端

在主仓库根目录使用 PowerShell 执行：

```powershell
$env:VERSION = "4.17.0-dream"

cd assets

yarn install --network-timeout 1000000
yarn version --new-version $env:VERSION --no-git-tag-version
yarn run build

cd ..
tar -a -cf application\statics\assets.zip assets/build
tar -xOf application/statics/assets.zip assets/build/version.json
```

如果提示没有 `yarn`，先执行：

```powershell
corepack enable
corepack prepare yarn@1.22.22 --activate
```

如果 `corepack` 不可用，可以改用：

```powershell
npm install -g yarn
```

构建成功后应生成：

```text
application/statics/assets.zip
```

注意：压缩包内需要保留 `assets/build` 这一层路径。确认最后一条命令输出的 `version` 与目标版本一致，再构建后端。`assets.zip` 是被 Git 忽略的构建产物，不需要提交。

## Windows 下构建 Linux amd64 后端

前端已经构建好以后，在主仓库根目录执行：

```powershell
$env:GOOS = "linux"
$env:GOARCH = "amd64"
$env:CGO_ENABLED = "0"

$env:COMMIT_SHA = git rev-parse --short HEAD
$env:VERSION = "4.17.0-dream"

go build -a -o cloudreve `
  -ldflags "-s -w -X github.com/cloudreve/Cloudreve/v4/application/constants.BackendVersion=$env:VERSION -X github.com/cloudreve/Cloudreve/v4/application/constants.LastCommit=$env:COMMIT_SHA"
```

输出文件：

```text
cloudreve
```

这个文件就是 Linux amd64 可执行文件。

两种平台的 `-ldflags -X` 都应使用 `go.mod` 中的 Go 模块路径 `github.com/cloudreve/Cloudreve/v4`，不改成 fork 的 Git 地址。

## Windows 下构建 Windows amd64 后端

```powershell
$env:GOOS = "windows"
$env:GOARCH = "amd64"
$env:CGO_ENABLED = "0"

$env:COMMIT_SHA = git rev-parse --short HEAD
$env:VERSION = "4.17.0-dream"

go build -a -o cloudreve.exe `
  -ldflags "-s -w -X github.com/cloudreve/Cloudreve/v4/application/constants.BackendVersion=$env:VERSION -X github.com/cloudreve/Cloudreve/v4/application/constants.LastCommit=$env:COMMIT_SHA"
```

输出文件：

```text
cloudreve.exe
```

## 子模块提交顺序

`assets` 是独立的前端 Git 仓库。主仓库记录的是前端提交，不会自动跟随前端分支头。先在主仓库根目录对照两个提交：

```powershell
git ls-tree HEAD assets
git -C assets rev-parse --short HEAD
```

如果修改了前端，先提交并推送前端，再让主仓库记录新的前端提交。例如仅改 Blob 表单时：

```powershell
cd assets
git status
git add src/component/Admin/StoragePolicy/EditStoragePolicy/FormSections/StorageAndUploadSection.tsx
git commit -m "移除 Blob 名称唯一变量限制"
git push -u origin dream-4.17.0
cd ..
```

然后在主仓库提交子模块指针；有其他主仓库改动时，另按本次范围暂存。

```powershell
git status
git add assets
git commit -m "更新前端子模块指针"
git push -u origin dream-4.17.0
```

## 常见问题

### 1. 为什么 VS Code 里有两个 Git 仓库？

因为 `assets` 是 Git 子模块，是独立的前端仓库。主仓库和前端仓库需要分别提交。

```text
cloudreve
└── assets
    └── cloudreve-frontend
```

### 2. 后端版本和前端版本不一致怎么办？

例如出现 `Current 4.0.0-next, Desired: 4.17.0-dream`，表示当前实际加载的前端资源仍是旧版。这个检查会记录错误，但本身不返回启动错误。

1. 先看部署端日志有没有 `Folder with ... already exists`。服务优先读取实际数据目录的 `data/statics`（默认相对于可执行文件所在目录，使用 `--use-working-dir` 时相对于工作目录）；存在时，先检查该目录中的 `version.json`。确认不再需要该目录覆盖内嵌资源后，可以先停服务、备份或改名该目录，再重启；保留自定义资源，别删除整个 `data` 或源码里的 `application/statics`。
2. 如果没有外部覆盖，检查构建机上的 `assets/build/version.json` 和包内的同一文件（使用上面的 `tar -xOf`）。包版本不对就用 `4.17.0-dream` 重建前端和压缩包，然后重新构建并部署后端。
3. 如果本地 ZIP 已正确，服务器仍报旧版本，核对服务启动命令指向的二进制是否已替换，且进程确实已重启。仅替换构建机上的 ZIP 不会更新已经编译的二进制。

### 3. 只改后端，需要重新构建前端吗？

如果 `application/statics/assets.zip` 已经存在，并且版本号和后端一致，不需要重新构建前端。

### 4. 只改前端，需要重新构建后端吗？

需要。前端会被打包成 `application/statics/assets.zip`，后端二进制会嵌入这个 zip 文件，所以前端改动后需要重新构建后端。
