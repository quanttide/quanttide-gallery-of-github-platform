# 初始化第二大脑的资产子模块

本案例记录一次按惯例为第二大脑新增资产子模块的过程：惯例没有成文规范，命名与坐标从既有子模块反推，创建云端仓库后挂载到标准坐标。案例沉淀为四阶段流程与验证清单。

## 需求

### 情境

第二大脑 `adapters/quanttide-github` 需要新增三类资产：报告、案例集、实验室。适配轴的建仓惯例散落在同轴仓库的实物里——`quanttide-pi` 有 context / profile / handbook / gallery，`quanttide-feishu` 有 lab / gallery。约束有四：仓库保持公开；命名与坐标遵守本仓既有 `.gitmodules`；云端仓库必须可克隆（须有初始提交）；子仓与父仓各自提交推送，互不混入对方变更。

### 难点

难点有四：

- 命名依赖两个名字，都不在父仓任何文档里：长名（英文名）只能从既有 `-of-{长名}` 子模块反推（`quanttide-context-of-github-platform` → `github-platform`），短名取自第二大脑仓名（`quanttide-github` → `github`）；
- 云端空仓库与子模块不兼容：`gh repo create` 默认不写初始提交，子模块指针指向空提交，别人克隆后目录是空的；
- 模板按资产类型不同：lab 与 gallery 用 README + CHANGELOG + LICENSE，report 空仓只有 README + LICENSE，须逐类型对照实例；
- 许可不统一：lab 用 Apache 2.0，asset 用 CC BY 4.0；实例之间有分歧时无从反推，只能问人。

## 意图

### 自然语言规则

1. 命名封闭：仓库名由领域名与资产类型唯一确定，分两套风格，各自内部统一——**文档仓**（context、journal、intention、report、gallery、handbook 等）取长名 `quanttide-{类型}-of-{英文名}`；**代码仓**（platform、toolkit、laboratory）取短名 `qtcloud-{短名}`、`quanttide-{短名}-toolkit`、`quanttide-{短名}-lab`。长名与短名是不同的名字（如 `github-platform` 对 `github`），均取自本仓既有命名，不从中文直译；
2. 坐标固定：**文档仓**挂固定短名目录（`data/report`、`docs/gallery`…）；**代码仓**挂 `{标准目录}/{仓库原名}`（如 `examples/quanttide-github-lab`）——直接用原名，避免独立开工作区时重名，`examples/default` 这类固定别名已废弃；
3. 非空：云端仓库必须有初始提交，子模块指针才可解析；
4. 最小模板：只建 README（按类型加 CHANGELOG）+ LICENSE，不预铺目录、不预建空仓库；
5. 图层提交：子仓先提交推送，父仓再提交指针与 CHANGELOG；
6. 有疑必问：命名或许可有歧义时先问，不臆测。

### 形式化规则

```text
对象：B 第二大脑，S 领域短名，E 领域英文名，t 资产类型，U 云端仓库，a 挂载坐标

命名（两套风格，各自内部统一）
  文档仓  t ∈ Tdoc  = {context, journal, intention, report, gallery, handbook, …}
      repo(t) = quanttide-{t}-of-{E}
  代码仓  t ∈ Tcode = {platform, toolkit, laboratory}
      repo(platform)   = qtcloud-{S}
      repo(toolkit)    = quanttide-{S}-toolkit
      repo(laboratory) = quanttide-{S}-lab

  取值：E 是长名，多为两个单词或一个长词（如 github-platform），取自已有的
        `-of-{E}` 子模块；S 是短名，为一个单词或缩写（如 github），取自领域
        （与第二大脑仓名 quanttide-{S}）。两者不是同一个名字，不可互换。

坐标
  mount(t) : t ↦ 固定路径
      文档仓 → 固定短名目录
          context / journal / intention → data/{t}
          report → data/report   gallery → docs/gallery   handbook → docs/handbook
      代码仓 → {标准目录}/{仓库原名}（直接用原名，避免独立开工作区时重名）
          platform   → apps/qtcloud-{S}
          toolkit    → packages/quanttide-{S}-toolkit
          laboratory → examples/quanttide-{S}-lab

操作
  create(U)          建云端仓库并写初始文件
  mount(U, B) @ a    在 B 的坐标 a 挂载子模块 U
  register(B)        父仓登记 .gitmodules 并提交指针

不变式（操作后必须全部成立）
  I1 命名：name(U) = repo(t) ∧（文档仓取 E，自 submodules(B)；代码仓取 S，自领域）
  I2 坐标：gitlink(B, U) 位于 mount(t)
  I3 非空：commits(U) ≠ ∅
  I4 可解析：submodule_status(B, U) 指向提交 ∧ 克隆后目录非空
  I5 图层：commits(U) 先于 指针提交(B) ∧ 两侧均已推送
```

## 规格

### 自然语言描述

实现为四阶段流水线，每阶段以状态断言收尾：

1. 反推：读本仓 `.gitmodules` 得长名 `E`（自既有 `-of-{E}` 子模块），读第二大脑仓名 `quanttide-{S}` 得短名 `S`；对照同轴实例定坐标 `mount(t)` 与模板；断言 `E`、`S`、`t`、`a` 已知；
2. 建仓：以 `gh repo create --add-readme` 建云端仓库（`--add-readme` 提供初始提交）；断言仓库存在且至少一个提交；
3. 挂载：`git submodule add` 到坐标 `a`，写 README / LICENSE（按类型加 CHANGELOG）；断言 `git submodule status` 出现该条目；
4. 提交推送：子仓一个 `docs: 初始化…` 提交并推送，父仓一个 `feat: 注册…子模块` 提交（含 `.gitmodules`、指针、CHANGELOG）并推送。

### YAML 描述

机器可读的规格表示，与实现脚本逐阶段对应：

```yaml
meta:
  brain: adapters/quanttide-github
  asset_type: report
  domain_short: github                     # 短名，取自第二大脑仓名 quanttide-github
  domain_english: github-platform          # 长名，取自 quanttide-context-of-github-platform
  repo: quanttide-report-of-github-platform
  mount: data/report
  org: quanttide

steps:
  - id: derive          # 1 反推
    action: 从既有子模块取长名、从第二大脑名取短名，定坐标与模板
    commands:
      - grep -o 'quanttide-context-of-[a-z-]*' {brain}/.gitmodules   # → 长名 github-platform
      - basename {brain} | sed 's/^quanttide-//'                     # → 短名 github
    assert:
      - domain_english = github-platform
      - domain_short = github

  - id: create          # 2 建仓
    action: 建云端仓库，带初始提交
    commands:
      - gh repo create {org}/{repo} --public --add-readme
    assert:
      - commits({repo}) ≠ ∅              # 对应不变式 I3

  - id: mount           # 3 挂载
    action: 挂载子模块到坐标并写最小模板
    commands:
      - git -C {brain} submodule add https://github.com/{org}/{repo}.git {mount}
    assert:
      - submodule_status({brain}) 包含 {mount}
        # 对应不变式 I2、I4

  - id: publish         # 4 提交推送
    action: 子仓、父仓各一个提交并推送
    commits:
      - repo: child
        message: "docs: 初始化 GitHub 平台报告（README / LICENSE）"
      - repo: parent
        message: "feat: 注册报告子模块（data/report → quanttide-report-of-github-platform）"
    assert:
      - 不变式 I5 成立 ∧ 两侧已推送
```

> 提示：`git submodule add` 一个没有提交的云端空仓库会失败（unborn branch，`fatal: 您位于一个尚未初始化的分支`）；建仓务必带 `--add-readme`，或进目录手动提交并 `git push -u origin main` 后再重跑挂载。

## 实现

按上述规格参数化后的实现，取 `report` 一例：

```bash
#!/usr/bin/env bash
# 初始化第二大脑的资产子模块：反推 → 建仓 → 挂载 → 提交推送
set -euo pipefail

B=~/repos/quanttide/adapters/quanttide-github   # 第二大脑
ORG=quanttide
E=github-platform                                # 长名（英文名），取自既有 -of- 子模块
T=report                                         # 资产类型；report 属文档仓
# 代码仓（platform/toolkit/laboratory）不用 E，改用短名 S=github，命名式不同
REPO="quanttide-$T-of-$E"                         # → quanttide-report-of-github-platform
MOUNT=data/report                                # 坐标，由类型确定

# 阶段 1：从既有子模块反推长名（英文名）
grep -o 'quanttide-context-of-[a-z-]*' "$B/.gitmodules"     # → github-platform

# 阶段 2：建云端仓库，带初始提交（否则子模块指针为空）
gh repo create "$ORG/$REPO" --public --description "量潮 GitHub 平台报告" --add-readme

# 阶段 3：挂载到标准坐标并写最小模板
cd "$B"
git submodule add "https://github.com/$ORG/$REPO.git" "$MOUNT"
cp LICENSE "$MOUNT/LICENSE"
# 写 "$MOUNT/README.md"

# 阶段 4：子仓提交推送，父仓提交指针与 CHANGELOG
git -C "$MOUNT" add -A
git -C "$MOUNT" commit -m "docs: 初始化 GitHub 平台报告（README / LICENSE）"
git -C "$MOUNT" push
git -C "$B" add .gitmodules "$MOUNT" README.md CHANGELOG.md
git -C "$B" commit -m "feat: 注册报告子模块（$MOUNT → $REPO）"
git -C "$B" push
```

案例执行结果：在同一第二大脑上一次建成 lab、gallery、report 三类资产子模块。子仓根提交分别为 `quanttide-github-lab` `e3738e3`、`quanttide-gallery-of-github-platform` `487df3c`、`quanttide-report-of-github-platform` `1b0de4d`；父仓登记提交 `e050c53` / `4fb778e` / `1d8aadf`，不变式 I1 至 I5 全部核对通过。lab 的许可是唯一的例外（Apache 2.0，由人裁定），不满足「从实例反推」这条。
