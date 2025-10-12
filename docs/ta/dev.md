# 助教开发手册

本课程的开发在 ZJU Git 上进行，GitHub 定期手工同步。

## 仓库列表

- （公开）实验代码发布仓库 [ZJU-OS/code](https://git.zju.edu.cn/os/code)

    用于发布实验代码。

- （私有）完整代码仓库 [ZJU-OS/code-private](https://git.zju.edu.cn/os/code-private)

    `main` 分支存放实现了所有实验功能的代码，持续进行整体代码的维护和改进。

!!! tip "GitHub 的一些缺陷"

    - GitHub 不支持 Public 仓库的 Private Fork，**而我们希望从私有仓库到代码发布能走 PR 流程，确保参与的助教知道发布的代码发生了什么变更**。
    - GitHub Pull Request 不能同时支持下面两个要求：

        - 不生成 Merge Commit
        - 保留 Commit 的 Hash

        这给多仓多分支开发造成了很多麻烦。在本地执行 `git merge` 时，如果可以 Fast-forward 则能同时满足上面两个要求，这样比较优雅，能够保持[线性历史](https://stackoverflow.com/questions/20348629/what-are-the-advantages-of-keeping-linear-history-in-git)。但 GitHub 实际上执行的更像是 `git merge --no-ff`，见 [git - GitHub: Commit is changed after merge - Stack Overflow](https://stackoverflow.com/questions/52849531/github-commit-is-changed-after-merge)。即使用 Merge 的方式合并 PR，也会产生一个多余的 Merge Commit，需要 Sync Fork。

    GitLab 可以满足上面两个要求。

## 发布流程

下面从每学期初开始介绍流程。示意图中红色的节点是携带全部代码的提交，不应当公开。绿色节点是安全的，可以公开的提交。

![workflow.drawio](workflow.drawio)

- 创建本学期的发布分支 `fa25-release`：挖空 `main` 的代码，创建不带历史记录的分支推送到学生仓库

    ```shell
    git clone git@github.com:/ZJU-OS/code-private.git
    cd code-private
    git remote rename origin code-private
    git checkout -b fa25-release-clean main
    # 挖空代码
    git commit -am "fa25: init"
    git checkout --orphan fa25-release
    git commit -am "fa25: init"
    git push -u code-private fa25-release
    ```

- 创建空的发布仓库，推送第一个基础分支 `lab0`（仅此时不需要走 MR 流程）：

    ```shell
    git remote add code git@github.com:/ZJU-OS/code.git
    git checkout -b lab0 fa25-release
    git push code lab0
    ```

- `main` 进行其他开发工作
- 发布 `labN`：
    - 以 `fa25-release` 为基础（此时已经发布了 `lab(N-1)`）,从 `main` 选择需要的代码进行合并，挖空不需要的代码，达到 `labN` 的进度
    - 创建 GitLab Merge Request 合并到学生仓库的 `labN` 分支

    ```shell
    git checkout fa25-release
    # 以当前工作目录为基础，比较 main 分支新增哪些代码
    git diff -R --compact-summary main --
    # 从 code-private/main 合并代码到 code-private/fa25-release
    # 可以用 git merge 或 git checkout
    git merge --squash --no-commit main
    git checkout main -- <需要的文件>
    # 手工选择需要合并的文件，挖空学生完成的代码
    # 可以分多次提交，更加清晰地添加文件
    git commit -am "labN: release"
    # GitLab 上创建 Merge Request
    # 从 code-private/fa25-release 合并到 code/labN
    ```

建议在 `.gitconfig` 中配置 `difftool`，方便用 VSCode 查看代码变更：

```ini title="~/.gitconfig"
[diff]
    tool = vscode
[difftool "vscode"]
    cmd = code --wait --diff $LOCAL $REMOTE
```

配置后，可以用 `git difftool` 代替 `git diff` 查看代码变更：

```shell
git difftool main --
```

上面这行命令会对所有有差异的文件逐个打开 VSCode 的对比界面，可以利用 VSCode 的差异合并功能方便地处理变更。完成一对文件的对比后，关闭对应的 VSCode 窗格即可进入下一对文件的对比。

## 开发日志

### Lab2

遗留待完善：

- 将调度算法升级到 CFS
- 测例计时没区分进程时间和内核时间（就是 schedule 的开销）
- 进程阻塞和唤醒机制
