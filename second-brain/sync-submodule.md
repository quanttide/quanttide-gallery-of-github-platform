# 同步子模块到云端

本案例记录一次把第二大脑整棵子模块树同步到云端的操作：树里散着未推送的提交、未提交的指针与落后远端的仓，要求全部提交推送完毕。案例沉淀为四阶段流程与验证清单。

## 需求

### 情境

一棵根仓库加四层子模块的树（根 → 轴目录 → 领域/资产仓 → 仓内嵌套子仓），实测规模 667 个仓，其中 22 个有未推送的提交、3 个落后远端、3 处未提交改动（含未提交的指针）。要求是「全部提交到云端」。约束有四：各仓独立提交推送，父仓只收指针；提交信息与变更内容一致，不把别处既有脏指针扫进自己的提交；远端此刻仍在推进（别处可能刚推过同一仓）；过程中不阻塞使用者。

### 难点

难点有五：

- 判据不在推送输出里：`git push` 退出码为零、输出 `Everything up-to-date` 都可能对应「远端没动」，只有「本地 HEAD 等于远端 main」才是证据；
- 非快进不能靠重试：远端在别处前进过时，本仓提交与远端分叉，重试多少次都是被拒——本次 22 个待推仓里有 4 个是这种情况；
- 分离头：`git submodule update --init` 检出的是父仓 gitlink 记的那个提交，不是分支头，子模块默认就在游离头上；此时提交落在游离头，推送推的是没动的 `main`（假成功）或直接非快进被拒；
- 顺序不可倒：父仓指针先于子仓推上去，父仓就指向一个远端不存在的提交（悬空指针）；而文件常落在嵌套子仓（`docs/gallery`、`data/context` 各自是独立仓），只按一层遍历会漏；
- 观测本身会假死：串行遍历加每仓「推送 + fetch + 校验」三重超时，单仓最坏 90+40+40 秒，几十个仓排下来长时间没有任何输出，看起来像卡死（本次第一版脚本如此）。

## 意图

### 自然语言规则

1. 判据为证：同步成立 = 该仓本地 HEAD 等于远端 `main` 的 sha；退出码、`Everything up-to-date`、`Already up to date` 都不作为判据；
2. 顺序自下而上：先推叶子仓，再让中间仓收指针，最后根仓收指针；父仓的指针提交必须晚于子仓的推送；
3. 先对齐再推送：推送前先 `fetch`，落后就快进，分叉就变基到远端再推；非快进不重试；
4. 分支归位：分离头先 `git branch -f main HEAD && git checkout main`，再推送；
5. 只收已同步的指针：仅当子仓「无未推提交且无落后」时，才允许把它的 HEAD 收进父仓提交；
6. 归属先判：遍历到文件先判它归哪个仓（含嵌套子仓），不用一条 `git submodule foreach --recursive` 串命令驱动全部动作；
7. 不阻塞：清点与推送并行进行、放后台跑、完成即通知，过程中不占住使用者；
8. 报告为对账：收尾输出全树的 `AHEAD`（`origin/main..HEAD` 非空）与 `DIRTY`（工作区非空）两列，两列皆空才算完。

### 形式化规则

```text
对象：T 子模块树，R ∈ T 仓，P(R) R 的父仓，C(P) P 的子仓集合
      head(R) 本地 HEAD，remote(R) 远端 main 的 sha

操作
  align(R)      fetch 后快进或变基，使 behind(R) = 0
  push(R)       推送 main 至远端
  collect(P)    把子仓 head 收进 P 的指针提交并推送

不变式（收尾时必须全部成立）
  I1 已推：∀ R：head(R) = remote(R)
  I2 对齐：∀ R：behind(R) = 0
  I3 指针一致：∀ R，∀ c ∈ C(R)：gitlink(R, c) = head(c) ∧ c 无脏指针
  I4 顺序：∀ c：推送(c) 先于 提交指针(P(c))
  I5 原子：每仓一条提交，提交信息与变更内容一致，不裹入非本次变更
  I6 零残留：AHEAD = ∅ ∧ DIRTY = ∅
```

## 规格

### 自然语言描述

实现为四阶段流水线，每阶段以状态断言收尾：

1. 清点：遍历全树，本地算出每仓的 `AHEAD`（`origin/main..HEAD` 条数）、`BEHIND`（`HEAD..origin/main` 条数）与 `DIRTY`（`git status --porcelain` 行数）；断言清单非空且仓数合理（本次 667，若骤降说明遍历根目录不对）；
2. 对齐推送：对每个 `AHEAD > 0` 的仓——分离头先归位 `main`，`fetch` 后落后则快进、分叉则变基，再推送并核对 `head == remote`；变基冲突则中止该仓、标注留人工；断言每个仓要么在同步集合内、要么带原因被标注；
3. 收指针：逐轮扫描父仓的脏子模块指针，只收 `AHEAD = 0 ∧ BEHIND = 0` 的子仓；每仓一个指针提交并推送；重复直到没有可收的指针（层数决定轮数，本树 4 层）；断言每轮收敛且不残留脏指针；
4. 对账：全树再算一遍 `AHEAD`/`DIRTY`；断言两列皆空，输出剩余问题清单（空清单即完成）。

### YAML 描述

机器可读的规格表示，与实现脚本逐阶段对应：

```yaml
# 同步子模块到云端：清点 → 对齐推送 → 收指针 → 对账
meta:
  root: ~/repos/quanttide                # 子树根仓库
  parallel: 4                            # 并行推送数
  net: "-c http.proxy= -c https.proxy= -c http.version=HTTP/1.1"   # 绕开本机常死的代理
  skip_dirs: [".dart_tool", "node_modules", ".venv", "target", "build"]

steps:
  - id: scan                             # 1 清点
    action: 遍历全树，本地算 AHEAD / BEHIND / DIRTY
    commands:
      - find . -name .git -prune ...     # 逐仓列出（跳过构建目录）
      - git -C {r} rev-list --count origin/main..HEAD    # AHEAD
      - git -C {r} status --porcelain                    # DIRTY
    assert:
      - 仓数 = 667（异常下降说明根目录取错，中止）

  - id: align-push                        # 2 对齐推送
    action: 分离头归位 → fetch → 快进/变基 → 推送 → 校验
    commands:
      - git -C {r} branch -f main HEAD && git -C {r} checkout main
      - git -C {r} fetch origin        # 落后则 merge --ff-only，分叉则 rebase origin/main
      - git -C {r} push origin main
      - git -C {r} ls-remote origin main
    assert:
      - head(r) = remote(r)              # 对应 I1；非快进不重试
      - 变基冲突 → 中止该仓并标注（不阻塞其余仓）

  - id: collect                           # 3 逐层收指针（重复至收敛）
    action: 只收「无未推且无落后」的子仓指针
    commands:
      - git -C {p} add {c} && git -C {p} commit -m "chore: 更新子模块指针"
      - git -C {p} push origin main
    assert:
      - 子仓已推（I4）∧ 父仓已推（I1）∧ 父仓无脏指针（I3）

  - id: audit                             # 4 对账
    action: 全树重算两列
    commands:
      - git -C {r} rev-list --count origin/main..HEAD
      - git -C {r} status --porcelain
    assert:
      - AHEAD = ∅ ∧ DIRTY = ∅            # 对应 I6
```

> 提示：遍历与校验一定要分开写。把「提交/推送」和「递归校验」塞进一条命令，既会撞超时，也会让失败点不可分辨；推送并行、校验一次性收尾，观测才有中间输出。

## 实现

按上述规格参数化后的实现骨架（已省略日志细节）：

```bash
#!/usr/bin/env bash
# 同步子模块到云端：清点 → 对齐推送 → 收指针 → 对账
set -u
R=~/repos/quanttide
NO="-c http.proxy= -c https.proxy= -c http.version=HTTP/1.1"
export GIT_TERMINAL_PROMPT=0 GIT_EDITOR=true      # 无凭据即失败，不留在等输入上

repos() { find "$R" -maxdepth 9 \( -name .dart_tool -o -name node_modules -o -name build \) -prune \
            -o -name .git -print | sed 's|/.git$||'; }

sync_repo() {  # 对齐 + 推送 + 校验，判据只有一条：head == remote
  d=$1
  [ -z "$(git -C "$d" branch --show-current)" ] && { git -C "$d" branch -f main HEAD; git -C "$d" checkout -q main; }
  git -C "$d" $NO fetch -q origin
  if ! git -C "$d" merge --ff-only origin/main >/dev/null 2>&1; then
    git -C "$d" rebase origin/main >/dev/null 2>&1 || { git -C "$d" rebase --abort; return 9; }
  fi
  git -C "$d" $NO push origin main >/dev/null 2>&1
  [ "$(git -C "$d" rev-parse HEAD)" = "$(git -C "$d" $NO ls-remote origin main | cut -f1)" ]
}

# 阶段 1–2：清点待推仓，并行对齐推送（4 路）
repos | while read -r rel; do
  a=$(git -C "$rel" rev-list --count origin/main..HEAD 2>/dev/null || echo 0)
  [ "$a" != 0 ] && echo "$rel"
done > /tmp/todo.txt
xargs -a /tmp/todo.txt -P 4 -I{} bash -c 'sync_repo "$@"' _ {} || echo "有仓未成，见日志"

# 阶段 3：逐层收「无未推且无落后」的子仓指针（重复到无指针可收）
for pass in 1 2 3 4 5 6; do
  found=0
  for p in $(repos); do
    for c in $(git -C "$p" status --porcelain | awk '{print $2}'); do
      [ -e "$p/$c/.git" ] || continue                      # 只处理子模块指针
      sa=$(git -C "$p/$c" rev-list --count origin/main..HEAD 2>/dev/null || echo 0)
      sb=$(git -C "$p/$c" rev-list --count HEAD..origin/main 2>/dev/null || echo 0)
      [ "$sa" = 0 ] && [ "$sb" = 0 ] || continue           # 子仓未同步则不收
      git -C "$p" add "$c"; found=1
    done
    git -C "$p" diff --cached --quiet || git -C "$p" commit -q -m "chore: 更新子模块指针（子仓已推送至云端）"
    git -C "$p" $NO push origin main >/dev/null 2>&1
  done
  [ "$found" = 0 ] && break
done

# 阶段 4：对账（两列皆空才算完）
for rel in $(repos); do
  a=$(git -C "$rel" rev-list --count origin/main..HEAD 2>/dev/null || echo 0); [ "$a" != 0 ] && echo "AHEAD $rel $a"
  s=$(git -C "$rel" status --porcelain | wc -l);                                [ "$s" != 0 ] && echo "DIRTY $rel $s"
done
```

案例执行结果：全族 667 个仓，清点出 22 个待推仓、3 个落后仓、3 处未提交改动；其中 4 个因远端已前进需先变基；收尾对账 `AHEAD`／`DIRTY` 两列均为空。
