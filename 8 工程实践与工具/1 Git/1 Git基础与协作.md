# 1 Git 基础与协作

Git 通过提交记录保存文件快照和变更历史。工作区、暂存区和仓库是三个不同状态：编辑文件产生工作区变更，`git add` 选择进入暂存区，`git commit` 将暂存内容写入本地历史。

## 1.1 常用流程

```bash
git status
git diff
git add -- path/to/file
git diff --cached
git commit -m "说明变更"
git fetch origin
git rebase origin/main
git push origin HEAD
```

提交应围绕单一目的，消息说明行为而非过程。推送前检查 staged diff、测试和敏感文件；不要用 `git add .` 无选择地纳入无关文件。

## 1.2 分支与冲突

分支适合隔离并行工作。合并或变基冲突时先理解双方业务意图，再解决文本冲突，运行测试后提交。不要用强制覆盖方式“解决”不理解的冲突；共享分支改写历史前必须确认协作影响。

## 1.3 恢复与安全

`git restore`、`git revert` 和 `git reset` 影响范围不同：恢复工作区、生成反向提交和移动本地历史不能混用。删除或覆盖前先确认目标、备份和未提交变更。密钥、令牌、个人数据和构建产物不应提交；发现泄露时要立即撤销和轮换。

