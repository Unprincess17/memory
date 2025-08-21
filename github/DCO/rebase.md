# Fix DCO commit message
```bash
git rebase HEAD~1 --signoff
git push --force-with-lease origin [分支名]
```

# 仅撤销 commit，保留修改的文件（工作区不变）
```bash
git reset --soft HEAD^
```

`HEAD^` 表示上一个 commit

修改的文件会保留在 暂存区（git add 后的状态），可重新修改后提交

# 如果已推送到远程仓库（强制覆盖）
```bash
git reset --soft HEAD^
git push --force-with-lease origin [分支名]
```

# 安全撤回（推荐用于已推送的提交）
```bash
git revert HEAD
```

这会创建一个新的 commit 来撤销上一次提交的更改

直接推送到远程即可：git push origin main