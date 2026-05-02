---
title: "使用 Github Template 时，如何同步源 Template 的上游更新？"
date: 2026-05-01
description: "简单操作一下。"
image: img/cover.png
tags:
    - Github Template
    - Git
categories:
    - Git
---

## 正文

之前写了一个 DN42 网络信息展示面板，MoonWX 希望后续用我的原仓库来搭建他自己的面板。因此我把原项目重构成了一个 Github Template，然后自己重新用这个 Template 创建了一个新的仓库来放置我自己的信息。我说保持复用性是对的。

但是问题来了，Github Template 的上游更新无法直接同步到下游仓库。也就是说，如果我在 Template 仓库里修复了一个 bug 或者添加了一个很好的新功能，下游仓库是无法自动获得这个更新的。

正好今天发现一个小前端问题，在 Template 仓库里修复了这个问题，下面借此实操一下同步。

首先你需要在本地的下游仓库里添加一个新的上游远程仓库，在我这里就是：

```bash
git remote add upstream https://github.com/SisypheOvO/DN42-Network-Info-Template.git
git remote -v # 验证添加是否成功
```

或者其 SSH 形式。接下来，就可以拉取更新了：

```bash
git fetch upstream
```

这会将上游 (upstream) 的所有新提交下载到本地，但不会自动修改代码。接下来有两种方式可以选择用于将更新合并到你自己的主分支。

### 一：git merge

创建一个合并提交来记录更新，能清晰地保留两个分支的历史，处理冲突时也相对直观安全。

```bash
git checkout main # 切换到自己的主分支
git merge upstream/main # 将上游的 main 分支合并到当前分支
# 此处处理可能出现的冲突 ...
git push origin main # 将合并后的更新推送到 GitHub
```

在合并过程中，可能会遇到冲突，而且第一次概率很大，因为作为 Template 使用源仓库时，多半还是要改一些配置和数据才能为己用的。如果出现了冲突，Git 会提示哪些文件存在冲突。之后如果需要查看哪些文件仍未解决冲突，可以使用：

```bash
git status
```

打开冲突文件，你会看到类似这样的标记：

```plaintext
<<<<<<< HEAD
（你当前分支的内容，即你自己的数据相关配置）
=======
（upstream/main 分支的内容，即模板的样式更新）
>>>>>>> upstream/main
```

手动编辑这些文件，决定保留哪些（可以是任意组合）。完成后，**要记得删除`<<<<<<<`、`=======`、`>>>>>>>`这些标记行**。然后，像普通提交一样，标记冲突已解决并完成合并：

```bash
git add .
git commit -m "Merge upstream's update and resolve conflicts" # 或者别的什么标题
```

### 二：git rebase

如果希望保持提交历史是一条整洁、线性的直线，可以使用 `git rebase`。它会将你的提交“重新播放”到上游更新的顶部。

```bash
git checkout main # 切换到自己的主分支
git rebase upstream/main # 执行变基操作
# 此处处理可能出现的冲突 ...
git push origin main --force # 变基后需要强制推送
```

变基过程中如果遇到冲突，Git 会暂停并提示（类似 merge）。解决步骤如下：

1. 同上手动编辑冲突文件，解决冲突。

2. ```bash
    git add <冲突文件>
    ```

3. ```bash
    git rebase --continue
    ```

4. **重复上述步骤，直到所有提交都成功应用**。如果过程中遇到困难，可以用 `git rebase --abort` 安全地终止变基操作，回到变基前的状态。

## 冷知识

如果你本地的 `git config user.email` 输出的配置不是认证邮箱，你的 commit 在 GitHub 上会显示为灰色默认头像，而且 commit 状态是 Unverified。

这时候如果你去 GitHub 的设置里把邮箱加上，你的 commit 就会从黄色的 Unverified 变回绿色的 Verified，这其实说明 commit 的 Verified 状态不是静态的。

同样这（应该）也适用于 `git config user.name`。
