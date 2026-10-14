# 特殊文件

量潮各仓库都带**同一套**特殊文件，文件名与用途一致；变的是**具体文本**——每类仓库各有各的写法。下面逐个文件给出文本：本类仓库（案例集）已有现成的直接采用，其余照此套用。

## README.md

```markdown
# <仓库名>

量潮<中文名>——<一句话定位：是什么、给谁看、与同域资产的分工>。

## 结构

| 路径 | 仓库 | 说明 |
| :-- | :-- | :-- |
| `<挂载路径>` | [<仓库>](<url>) | <承担什么> |

## 许可

[<许可>](LICENSE)
```

## AGENTS.md

```markdown
# Agent Instructions

<本仓给 AI 的额外规则。>

## 提交规则

每次 commit 以后自动 push。
```

## CONTRIBUTING.md

本类仓库现成的文本，直接采用：

```markdown
# 贡献指南

`quanttide-gallery-of-github-platform` 是 GitHub 平台的案例集。案例是写给人看的——合作方、应聘者、其他适配轴仓库——不是内部归档。

## 入库边界

- 只放可公开的案例：涉及凭证、私有仓库、未脱敏内部材料的不入库；公开的组织与仓库可以点名。
- **不写执行记录**：提交号、推送是否成功、执行日志、验证通过的流水账，都不进案例。案例回答「做过哪类事、教训是什么」，不回答「这次跑通了」。
- 结论要能脱离本次运行成立：某次运行的产物是快照，不当作规格。

## 组织方式

一个主题一个目录，如 `second-brain/`（第二大脑的资产子模块操作）；目录内一篇一个案例，文件名描述这条案例。

## 案例要精炼

一条案例写一类操作，不按实例累积。同一类操作在多个仓库各落地一次（如「初始化资产子模块」在 github、feishu 各做一遍），仍只写一条案例——换一个仓库不另加，实例的差异留在参数里，不进正文。删掉与结论无关的运行细节，留可复用的做法与教训。

## 案例结构

一篇案例按四节组织，与已收录的案例一致：

1. **需求**：情境与难点——为什么做、难在哪；
2. **意图**：自然语言规则与形式化规则——要满足什么；
3. **规格**：分几阶段、每阶段的断言，给自然语言描述与 YAML；
4. **实现**：可直接跑的脚本。

规则写在意图里，脚本只是其中一种实现。

## 命名

小写字母、连字符 `-` 分隔，如 `init-submodule.md`。

## 提交流程

本仓库是 `quanttide-github` 的子模块，改完逐层回到父仓库：

1. 子模块内提交推送：`git commit -m "docs: …"`、`git push`；
2. 更新父仓库指针：在 `quanttide-github` 里 `git add docs/gallery && git commit -m "chore: 更新案例集子模块指针"`、`git push`。

提交信息用 Conventional Commits（`docs:` / `fix:` / `chore:`）。

## 编写规范

遵循量潮文档格式标准：标题最多三级；代码块标注语言；同一概念全程同名；能少则少。
```

## CHANGELOG.md

```markdown
# CHANGELOG

所有显著变更都将记录在此文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/)，版本号遵循 [SemVer](https://semver.org/)。

## [Unreleased]

### Added

- <新增>
```

## .gitignore

```gitignore
# 构建产物
__pycache__/
*.pyc
node_modules/
target/
build/
dist/

# IDE 配置
.idea/
.vscode/

# 系统文件
.DS_Store
Thumbs.db

# Office 临时文件
~$*.docx
```

## LICENSE

采用标准全文：代码仓 [Apache 2.0](LICENSE)（首行 `Copyright <年份> QuantTide`），文档仓 [CC BY 4.0](LICENSE)。

## ROADMAP.md

```markdown
# ROADMAP

> 格式：Keep a Changelog + checkbox 任务清单。

## [0.2.0]

### Added

- [ ] <里程碑>

### Fixed

- [x] <已完成>
```

## TODO.md

```markdown
# TODO

## <模块>

- [ ] <动作>（<文件>）
- [ ] <动作>（<文件>）
```

## STATUS.md

```markdown
# STATUS

| 子模块 | 最新版本 |
| :-- | :-- |
| <路径> | <版本> |
```

## index.md

```markdown
# <门户>

按角色（业务 / 职能）组织链接，指向各子模块或文档。
```
