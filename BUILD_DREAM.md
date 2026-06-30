# Dream 定制版构建说明

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

前端静态资源包 `application/statics/assets.zip` 和后端二进制里的 `BackendVersion` 必须一致。

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
Remove-Item application\statics\assets.zip -Force -ErrorAction SilentlyContinue
tar -a -cf application\statics\assets.zip assets/build
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

注意：压缩包内需要保留 `assets/build` 这一层路径。

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

`assets` 是独立的前端 Git 仓库，所以如果修改了前端，需要先提交前端仓库。

```powershell
cd assets
git status
git add src/component/Admin/StoragePolicy/EditStoragePolicy/FormSections/StorageAndUploadSection.tsx
git commit -m "移除 Blob 名称唯一变量限制"
git push -u origin dream-4.17.0
cd ..
```

然后再提交主仓库。

```powershell
git status
git add .gitmodules assets application/migrator/policy.go inventory/migration.go service/admin/policy.go application/statics/assets.zip BUILD_DREAM.md
git commit -m "定制存储策略 Blob 路径规则"
git push -u origin dream-4.17.0
```

## 当前定制内容

- 前端不再要求 Blob 名称规则必须包含 `{uuid}`、`{randomkey8}` 或 `{randomkey16}`。
- 后端迁移器不再因为存储策略规则缺少随机变量而强制覆盖为默认随机文件名。
- 后端默认本地存储目录规则从 `uploads/{uid}/{path}` 改为 `cloudreve/{path}`。
- 推荐存储策略：

```text
目录规则：cloudreve/{path}
Blob 名称规则：{originname}
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

重新用同一个 `$env:VERSION` 构建前端和后端。

例如都使用：

```powershell
$env:VERSION = "4.17.0-dream"
```

### 3. 只改后端，需要重新构建前端吗？

如果 `application/statics/assets.zip` 已经存在，并且版本号和后端一致，不需要重新构建前端。

### 4. 只改前端，需要重新构建后端吗？

需要。前端会被打包成 `application/statics/assets.zip`，后端二进制会嵌入这个 zip 文件，所以前端改动后需要重新构建后端。
