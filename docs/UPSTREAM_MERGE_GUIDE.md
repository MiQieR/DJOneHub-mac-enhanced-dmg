# 上游版本合并流程

本文档记录如何将上游项目 [rogerbush007-a11y/DJOneHub-mac-enhanced](https://github.com/rogerbush007-a11y/DJOneHub-mac-enhanced) 的新版本合并到本 fork，并编译出"上游新版本 + 我们的标准 macOS 应用改造"安装包。

## 背景

- 本 fork 在 `v0.1.3-preview` 基础上做了**标准 macOS 应用改造**：`DJOneHub.app`（状态栏图标 + 拖拽 DMG 安装），改造提交见 `12c033f`。
- 上游后续版本（v0.1.4 起）只改 Go 后端代码、README、CHANGELOG，**不触碰**我们的 macOS 改造部分，因此合并通常很干净，只有 `README.md` 会冲突。
- 合并策略：`git merge upstream/main`，然后修复冲突、验证编译、跑测试、编译 DMG。

## 一、准备

```sh
# 1. 确保 upstream remote 存在
git remote -v
# 若缺失:
git remote add upstream https://github.com/rogerbush007-a11y/DJOneHub-mac-enhanced.git

# 2. 拉取上游最新代码与 tag
git fetch upstream --tags
```

查看上游新增了什么：

```sh
# 最近的上游提交（含 tag）
git log --oneline upstream/main | head -10

# 上次合并点之后的新增提交。上次合并点 = 上一次 merge 的父提交（上游侧），
# 例如 v0.1.5 合并是 9644f17，v0.1.7 合并是 0bae842，以实际为准:
git log --oneline <上次上游提交SHA>..upstream/main

# 改动概览（排除已 vendored 的 third_party）
git diff <上次上游提交SHA> upstream/main --stat -- . ':(exclude)third_party/**'
```

## 二、合并

```sh
# 在工作树干净的前提下
git merge upstream/main --no-edit
```

预期结果：
- 若 `README.md` 冲突 → 按下面第三节处理。
- 其它文件（`cmd/`、`internal/`、`CHANGELOG.md` 等）通常自动合并，无需手工处理。

## 三、README.md 冲突处理

上游 README 维护的是"一键安装包 + 多版本下载列表"，我们维护的是"标准 app 安装包"。冲突必然出现在两处：

1. 顶部 `> [!IMPORTANT]` 版本公告块
2. `## 下载` 部分的下载列表

处理原则：**保留我们的标准 app 说明格式，只把版本号与特性描述更新为上游新版本**。参考模板：

```markdown
> [!IMPORTANT]
> 📦 **标准应用安装包已上传**：[Releases](https://github.com/rogerbush007-a11y/DJOneHub-mac-enhanced/releases) 提供标准 macOS DMG 安装包，双击打开后拖拽 `DJOneHub.app` 到 `Applications` 目录即可完成安装：
> - `DJOneHub-macOS-universal-v<新版本>.dmg`：基于上游 v<新版本>，支持 Apple Silicon 与 Intel Mac 通用双架构；<简要列出新版本新增特性>
```

`## 下载` 部分类似，列出 `DJOneHub-macOS-universal-v<新版本>.dmg` 并简述特性即可。

> 注意：合并后务必再次确认没有残留冲突标记：
> ```sh
> grep -n '^<<<<<<<\|^=======\|^>>>>>>>' README.md   # 应无输出
> ```

## 四、验证

```sh
# Go 编译 + vet
go build ./... && go vet ./cmd/...

# 测试（上游有时会新增 *_test.go）
go test ./cmd/djonehub-macos/

# 冲突标记检查
grep -rn '^<<<<<<<\|^>>>>>>>' cmd/ internal/ pkg/ 2>/dev/null
```

## 五、提交合并

```sh
git add README.md   # 以及其它冲突文件
git commit --no-edit
git log --oneline -3   # 确认合并提交已生成
```

## 六、编译 DMG

```sh
./scripts/build-dmg-universal.sh v<新版本>-preview
```

产物：`dist/DJOneHub-macOS-universal-v<新版本>-preview.dmg`

该脚本会依次：
1. 交叉编译 arm64 + x86_64 双架构 Go 后端与 libusb（`package-macos-universal.sh`）
2. 编译 Universal 版 Swift 通知助手与启动器
3. 组装 `DJOneHub.app` 并代码签名（`create-app-bundle.sh`）
4. 用 `hdiutil` 生成 DMG 并校验

## 七、验证产物（可选但推荐）

```sh
DMG=dist/DJOneHub-macOS-universal-v<新版本>-preview.dmg
hdiutil attach "$DMG" -nobrowse -quiet -mountpoint /tmp/djoh-check
APP=/tmp/djoh-check/DJOneHub.app
# 双架构检查
file "$APP/Contents/Resources/djonehub-macos"
# 签名检查
codesign --verify --deep --strict "$APP" && echo "SIGN OK"
# 新特性是否已编入二进制（用新版本源码里的函数名/日志关键词）
strings "$APP/Contents/Resources/djonehub-macos" | grep -c "<新特性关键词>"
hdiutil detach /tmp/djoh-check -quiet
```

## 八、推送

```sh
git push origin main
```

## 历史参考

| 版本 | 合并提交 | 上次上游 SHA | 新增特性 |
| --- | --- | --- | --- |
| v0.1.5 | `af9bd04` | `0bae842`（v0.1.5 即该点） | Windows 实验版、4G DHCP 自动续租、启动时 DHCP 修复 |
| v0.1.7 | `9b566f8` | `9644f17`（v0.1.5） | 信号自检与自动找回、USB 打开超时保护、自动创建/启用 4G 网卡服务 |

下次合并时，把"上次上游 SHA"更新为 `upstream/main` 当前指向即可。
