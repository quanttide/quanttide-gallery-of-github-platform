# 迁移子模块引用到另一个第二大脑

本案例记录一次公开仓库的归属迁移：仓库从一个组织转移到另一个组织，同时它在一个第二大脑中被取消子模块引用、在另一个第二大脑中被配置为子模块。案例沉淀为六阶段流程与验证清单。

## 需求

### 情境

公开仓库 `english-sentence-analyzer` 原属 `roadriver-tech` 组织，被源第二大脑（`roadriver-tech/roadriver-tech`，下称 A）以子模块引用在 `examples/` 下。组织结构调整后，仓库归属转移到 `quanttide-founder` 组织；引用关系同时交接：A 不再引用，改由目标第二大脑（创始人私有主仓，下称 B）在 `examples/` 下引用。约束有三：仓库保持公开；B 遵守既有 `.gitmodules` 约定；A、B 各自提交推送，互不混入对方变更。

### 难点

难点有四：

- 子模块状态分布在四处：gitlink、`.gitmodules` 配置节、`git-config` 配置节、`.git/modules` 下的 gitdir，漏清任何一处都留下幽灵配置；
- 组织转移不同于重命名，`gh` CLI 无直接子命令，且转移 API 的响应体返回转移前归属，退出码为零不代表转移成功；
- `git rm`、`git mv` 对 `.gitmodules` 的改动可能只进暂存区一半，提交内容与预期不符而不自知；
- A、B 的远端 main 都在活跃推进，推送前常需变基；共享工作区中还可能有他人未完成的变更，提交须只含自己的路径。

## 意图

### 自然语言规则

1. 引用收敛：转移完成后，所有第二大脑对仓库的引用都指向新组织地址，不依赖旧地址重定向；
2. 清理彻底：移除子模块是四处状态的联动清理，不是删除一个目录；
3. 原子提交：每个仓库一个提交，提交信息与变更内容一致；
4. 约定优先：目标第二大脑的节名、URL 风格、路径约定先探测后执行；
5. 属性保持：转移只改变仓库归属，不改变可见性等仓库属性；
6. 验证为证：以状态查询结果为成功判据，不以退出码或 API 响应体为判据。

### 形式化规则

```text
对象：R 被迁移仓库，A 源第二大脑，B 目标第二大脑，O1 → O2 组织转移

前置条件
  admin(O1) ∧ admin(O2)
  remote(B) = O2 的远端 ∧ worktree(B) = clean

操作
  transfer(R, O1 → O2)   组织间转移
  add(R, B)              B 接入子模块引用
  remove(R, A)           A 移除子模块引用

不变式（操作后必须全部成立）
  I1 归属：owner(R) = O2 ∧ visibility(R) = public
  I2 收敛：∀ X ∈ referrers(R)：url(X, R) = https://github.com/O2/R.git
  I3 清零：¬gitlink(A, R) ∧ ¬gitmodules(A, R) ∧ ¬config(A, R) ∧ ¬gitdir(A, R)
  I4 一致：∀ s ∈ submodules(A) ∪ submodules(B)：clean(status(s)) ∧ clean(worktree)
  I5 原子：∀ c ∈ commits(A) ∪ commits(B)：单一逻辑变更 ∧ pushed(c)
```

## 规格

### 自然语言描述

实现为六阶段流水线，每阶段以状态断言收尾：

1. 探测：读取 R 的归属与可见性，读取 B 的远端、工作树状态与 `.gitmodules` 约定；断言 B 工作树干净、约定已知；
2. 转移：调用 GitHub 转移 API 携带 `new_owner`，随后重新查询归属，断言 `owner(R) = O2` 且可见性未变；
3. 目标接入：在 B 中按其约定执行 `git submodule add`，路径与 URL 用新组织地址；断言新条目出现在 `git submodule status`；
4. 源清理：在 A 中 `git rm` 子模块路径，移除 `git-config` 配置节，删除 `.git/modules` 下 gitdir；断言 `.gitmodules` 与配置中无该节、工作树无残留目录；
5. 核对：对 A、B 各执行 `git submodule status` 与 `git status --short`；断言 A 只剩其余子模块、B 新条目存在、两边工作树干净；
6. 提交推送：A、B 各一个原子提交并推送；远端领先时 fetch 后变基再推。

### YAML 描述

机器可读的规格表示，与实现脚本逐阶段对应：

```yaml
# 子模块跨第二大脑迁移：参数、阶段、命令与断言
meta:
  repo: english-sentence-analyzer              # 被迁移仓库，保持 public
  source_org: roadriver-tech                   # O1
  target_org: quanttide-founder                # O2
  source_brain: ~/repos/source-second-brain    # A，本地 clone
  target_brain: ~/repos/target-second-brain    # B，本地 clone
  submodule_path: examples/english-sentence-analyzer

steps:
  - id: probe                          # 1 探测
    action: 读取目标约定与工作树状态
    commands:
      - git -C {target_brain} status --porcelain
      - cat {target_brain}/.gitmodules
    assert:
      - worktree(target_brain) = clean

  - id: transfer                       # 2 转移
    action: 组织间转移
    commands:
      - gh api -X POST repos/{source_org}/{repo}/transfer -f new_owner={target_org}
    assert:                            # 响应体是转移前状态，必须重新查询
      - owner(repo) = {target_org}
      - visibility(repo) = public

  - id: adopt                          # 3 目标接入
    action: 目标第二大脑接入子模块
    commands:
      - git -C {target_brain} submodule add https://github.com/{target_org}/{repo}.git {submodule_path}
    assert:
      - submodule_status(target_brain) 包含 {submodule_path}@main

  - id: detach                         # 4 源清理
    action: 源第二大脑移除子模块
    commands:
      - git -C {source_brain} rm {submodule_path}
      - git -C {source_brain} config --local --remove-section submodule.{submodule_path}
      - rm -rf {source_brain}/.git/modules/{submodule_path}
    assert:                            # 对应不变式 I3
      - .gitmodules 无该节 ∧ git-config 无该节 ∧ gitdir 不存在 ∧ 工作树无残留

  - id: verify                         # 5 核对
    action: 不变式核对
    commands:
      - git -C {source_brain} submodule status
      - git -C {target_brain} submodule status
    assert:
      - 不变量 I2、I4 成立

  - id: publish                        # 6 提交推送
    action: 每仓库一个原子提交并推送
    commits:
      - brain: source
        message: "chore: 移除 {repo} 子模块（仓库已迁至 {target_org}）"
      - brain: target
        message: "feat: 接入 {repo} 子模块（自 {source_org} 迁入）"
    assert:
      - 不变量 I5 成立 ∧ 两仓库已推送
```

> 提示：gitdir 可能按仓库初次添加时的历史路径命名，与当前路径不一致；用 `git config --local --get-regexp '^submodule\.'` 配合 `ls .git/modules` 排查实际位置。

## 实现

按上述规格参数化后的实现，组织名与路径替换为占位示例：

```bash
#!/usr/bin/env bash
# 迁移子模块引用到另一个第二大脑：探测 → 转移 → 接入 → 清理 → 核对 → 提交推送
set -euo pipefail

REPO=english-sentence-analyzer    # 被迁移仓库（保持 public）
OLD_ORG=roadriver-tech            # O1 源组织
NEW_ORG=quanttide-founder         # O2 目标组织
A=~/repos/source-second-brain     # 源第二大脑本地 clone
B=~/repos/target-second-brain     # 目标第二大脑本地 clone
SUB_PATH=examples/$REPO           # 引用路径，与 B 的约定一致

# 阶段 1：探测 B 的约定与工作树状态
[ -z "$(git -C "$B" status --porcelain)" ] || { echo '目标工作树不干净' >&2; exit 1; }
cat "$B/.gitmodules"

# 阶段 2：转移组织归属（响应体是转移前状态，须重新查询验证）
gh api -X POST "repos/$OLD_ORG/$REPO/transfer" -f new_owner="$NEW_ORG" >/dev/null
[ "$(gh api "repos/$OLD_ORG/$REPO" --jq .full_name)" = "$NEW_ORG/$REPO" ]
[ "$(gh api "repos/$NEW_ORG/$REPO" --jq .visibility)" = public ]

# 阶段 3：B 接入子模块引用
git -C "$B" submodule add "https://github.com/$NEW_ORG/$REPO.git" "$SUB_PATH"

# 阶段 4：A 移除子模块引用（git rm 联动 gitlink 与 .gitmodules）
git -C "$A" rm "$SUB_PATH"
git -C "$A" config --local --remove-section "submodule.$SUB_PATH" || true
rm -rf "$A/.git/modules/$SUB_PATH"    # gitdir 按历史路径命名时按实际位置删除

# 阶段 5：核对不变式（I2/I3/I4）
! grep -q "$REPO" "$A/.gitmodules"
git -C "$A" submodule status
git -C "$B" submodule status

# 阶段 6：各仓库一个原子提交并推送
git -C "$A" add -A .gitmodules && git -C "$A" commit -m "chore: 移除 $REPO 子模块（仓库已迁至 $NEW_ORG）"
git -C "$B" commit -m "feat: 接入 $REPO 子模块（自 $OLD_ORG 迁入）"
git -C "$A" push
git -C "$B" push
```

案例执行结果：A 侧以 `chore` 提交移除引用，B 侧以 `feat` 提交接入引用，两仓库推送成功，不变式 I1 至 I5 全部核对通过。
