# 对齐文件系统命名

本案例记录一条命名维护管道：当发现文件系统的命名与分类没有准确表达概念时，把该改的文件改名、把引用改到位。入口是一处用错的名字，出口是同名同义、跨仓库一致；案例沉淀为五条意图规则、一份机器可读的重命名规格与一套逐仓库提交的执行步骤。

## 需求

### 情境

组织第二大脑的 `tutorial/`、`handbook/`、`essay/` 三个文档仓库里，各有一份 `role/title.md`，讲的都是「职位」：手册给职位下定义，随笔讲模块化职位，教程以钱弘俶的官职为样本解读职位。同一套组织档案的另一处 `data/profile/orgs/*/structure/title.md`，记的是「职务」。

同一个文件名 `title.md`，在一处指职位，在另一处指职务。查证英文：职位是 position，职务是 title；title 的本义正是头衔、职务，属于「职务」一侧。命名与概念对不上，是要修的缺口。

### 难点

难点有三：

- 同名两义：`title.md` 同时指职位与职务，改错一处就把「职务」的记录误标成「职位」；
- 一词多仓：同一概念散在多个仓库，改名必须同改同步，否则一处 `position.md`、两处 `title.md`，读者无从判断是不是同一个东西；
- 引用成网：文件名被 `README.md`、`index.md`、`myst.yml` 目录项与正文相对链接引用，改名不止改文件名。

## 意图

### 自然语言规则

1. 概念先行：先确定概念及其边界与用词，职位是 position、职务是 title；
2. 一词一概念：同一个词不承担两个概念，同名两义优先消除；
3. 归类再改名：先判定每个文件承载哪个概念，再动名字，不得先改后判；
4. 改名连引用：改文件名的同时更新所有指向它的目录项与链接；
5. 跨仓库同步：同一概念在不同仓库用同一个名字，各自提交推送，主仓库逐层更新指针。

### 形式化规则

```text
对象：文件 f（文件名 n(f)），概念 c，仓库集 R

前置条件
  n(f) 与承载的概念不符
  概念 c 的用词已确定

操作
  classify(f)   判定 f 承载的概念 c
  rename(f, c)  把 n(f) 改为 c 对应的名字
  relink(f)     更新所有引用 n(f) 的目录项与链接
  commit(R)     各仓库分别提交推送，主仓库逐层更新指针

不变式（改名后必须全部成立）
  I1 同名同义：同一概念在 R 内各处名字相同
  I2 一词一概念：任一名字只对应一个概念
  I3 引用完整：无悬空的目录项或链接
  I4 留痕：改名与引用更新进入各仓库的提交历史
  I5 指针一致：主仓库子模块指针指向各仓库最新提交
```

## 规格

### 自然语言描述

管道分五步，每步以前一步的产出为输入：

1. 定概念与用词：说清职位与职务的边界，选定英文名 position 与 title；
2. 归类：全仓搜索 `title.md`，逐一判定归属——`role/title.md` 归职位，`structure/title.md` 归职务；
3. 改名：职位的三处 `git mv role/title.md role/position.md`，职务的 `structure/title.md` 不动；
4. 更新引用：改各仓库 `README.md`、`index.md`、`myst.yml` 的目录项与正文相对链接；
5. 提交：三处改名分属不同仓库，各自提交推送，主仓库逐层更新指针。

两条不论怎么实现都成立：先归类后改名，不得先改后判；改名必须连引用一起改，不留悬空链接。

### YAML 描述

机器可读的规格表示，与上面五步一一对应。

```yaml
概念:
  职位: position
  职务: title

迁移:
  - 从: role/title.md
    到: role/position.md
    仓库: [tutorial, handbook, essay]   # 承载职位，改名
  - 从: structure/title.md
    到: structure/title.md              # 承载职务，不动
    仓库: [profile]

引用:
  更新: [README.md, index.md, myst.yml, 正文相对链接]

提交:
  单位: 仓库                            # 各仓库分别提交推送
  指针: 主仓库逐层更新子模块指针
```

> 提示：`structure/title.md` 承载职务，在本次对齐中不改名。

## 实现

管道逐文件 `git mv`、逐仓库提交，没有自动执行器；关键是把「判定—改名—引用」串成一次操作，三步的顺序不能乱。

```bash
# 1 归类：列出全部同名文件，逐一判定归属
git grep -l 'title.md'

# 2 改名：只改承载职位的三处；structure 下的 title.md 不动
git mv role/title.md role/position.md

# 3 更新引用：README.md、index.md、myst.yml 的目录项与正文相对链接

# 4 提交：各仓库分别提交推送，主仓库逐层更新指针
git -C <仓库> add -A && git -C <仓库> commit -m "refactor: role/title.md → role/position.md"
git -C <仓库> push
git add <子模块> && git commit -m "chore: 更新 <子模块> 指针" && git push
```

出口状态：三处 `role/title.md` 改名为 `role/position.md`，`structure/title.md` 保留，各仓库的目录项与链接更新到位，主仓库子模块指针逐层更新。
