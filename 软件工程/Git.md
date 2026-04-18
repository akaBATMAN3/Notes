# Git

## 1 我有个远程仓库pangu_cxy基于远程仓库pangu，我在本地有仓库pangu_cxy基于远程仓库pangu_cxy，我在本地完成修改后需要提交到远程仓库pangu_cxy，然后在网页端手动提交合并请求。我本地和远程仓库pangu_cxy一定是落后于远程仓库pangu的，我现在应该执行什么命令顺利完成提交与合并

你的流程很清晰：本地 `pangu_cxy` 基于远程 `pangu_cxy`，你需要将本地修改推送到远程 `pangu_cxy`，然后发起合并请求到 `pangu`。由于远程 `pangu` 已经更新（你本地 `pangu_cxy` 落后于它），为了保证合并请求能够顺利无冲突地合并，你需要在推送**前**将远程 `pangu` 的最新代码同步到本地 `pangu_cxy`。

推荐使用 `rebase` 来保持历史线性，避免后续合并时产生不必要的合并提交或被阻塞。

### ✅ 标准操作步骤

bash

```
# 1. 获取远程仓库最新状态（包括 pangu 和 pangu_cxy）
git fetch origin

# 2. 切换到本地的 pangu_cxy 分支
git checkout pangu_cxy

# 3. 将本地 pangu_cxy 分支变基到最新的 origin/pangu 上
git rebase origin/pangu

# 如果有冲突，手动解决后：
#   git add <冲突文件>
#   git rebase --continue
# 如果想放弃：git rebase --abort

# 4. 推送变基后的本地分支到远程 pangu_cxy（因为历史变了，需要强制推送）
git push --force-with-lease origin pangu_cxy

# 5. 在网页端创建合并请求：源分支 = pangu_cxy，目标分支 = pangu
```



### 🔁 如果你不想用 rebase（不想强制推送）

可以使用 `merge` 方式：

bash

```
git fetch origin
git checkout pangu_cxy
git merge origin/pangu   # 将 origin/pangu 合并到本地 pangu_cxy
git push origin pangu_cxy
```



这种方式会产生一个合并提交，不会改变已有的提交哈希，因此不需要强制推送。之后在网页端创建合并请求也是可以的，只是历史中会多一个分叉点。

两种方式都能顺利完成提交与合并，推荐使用 `rebase` 以获得更整洁的线性历史。