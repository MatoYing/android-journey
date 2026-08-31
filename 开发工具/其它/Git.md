>Git：[https://juejin.cn/post/7037317549079396360](https://juejin.cn/post/7037317549079396360)
>Gerrit：[https://gerrit.cloudera.org/Documentation/intro-user.html](https://gerrit.cloudera.org/Documentation/intro-user.html)
#### 下拉遇到冲突
这种场景下，会有两种情况：
1）本地没有未提交修改 → pull 后会直接合并远程代码，如果有冲突就直接进入冲突状态让你解决。
```shell
git status # 看哪些文件有冲突
# 手动编辑文件解决冲突，删除<<、>>这些图标；也可以通过IDE的可视化方式
git add <file> # 标记冲突已解决
```
2）有未提交的本地修改 → pull 后会拒绝合并，提示先 commit/stash，因为合并可能会覆盖你的修改。
```shell
git stash
git pull
git pop
# 这种也是可能遇到冲突的，需要去解冲突
```
为什么未提交的情况需要呢么做？不能直接像上面一样解冲突吗？
因为已经提交的，是可以还原的。这种未提交的，一旦它覆盖了你工作区的修改，你就失去了还原机会，所以 Git 要求你 stash 或者 commit。
#### 新开发一个任务合理的步骤
为什么我经常下拉代码后，本地拉下来一坨别人的代码，并且让我解决冲突？
![[../../assets/Pasted image 20260829202354.png]]
上面的是我本地的 hmi_development_release2，下面是远程的，可以发现提交记录的顺序是不一样的。
理论上：
```
本地：A -> B
远程：A -> B -> C -> D
我pull后，本地 A -> B -> C -> D
```
但现在：
```
本地：A -> B
远程：A -> C -> B -> D
这样我pull，就不是Fast-Forward（快进合并），就会有问题，就会把C、B、E拉下来让你合并
        /---B'            （本地的B，和远程的B的哈希不同）
A------M                  （合并提交M）
        \---C---B---E     （远程）
```
但是这里应该无法做 “M” 这种操作的，因为 Gerrit 不允许你推送 merge commit，因为正常来说一个 Change 对应一个 commit，merge commit 会导致 review 混乱。
上面这种事情是会经常发生的，就算你提交前 pull，保证本地代码是最新的，但你无法保证远程仓库合并代码时的顺序，因为远程并不是你 pull 上去就合，你需要去等 review，这期间可能别的代码先合进去，从而造成我上面画的呢种情况。
所以最好的方式还是切出来一个分支进行开发，这样就可以保证在主分支上每次 pull 都是没问题的。
```shell
git checkout -b task_11111_hmi_development  # 命名时加上分支名，因为没法通过命令的方式知道这个分支来自于哪个分支，git pull时可能会拉错
# 分支上提交代码时
git push origin HEAD:refs/for/hmi_development
# refs/for/hmi_development：Gerrit特有的Review分支，你push时并不是直接push到远程的对应分支，而是先push到Gerrit的一个待审查区域，等待reviewer审核通过后，才会用rebase线性合并到hmi_development

# 等彻底测试验证通过后删除该分支
git branch -d <branch-name>
```
#### 分支切换与改动迁移
这种时候需要来回切换分支，两种情况：
1）在 A 分支做一些改动后，还没有 commit，这时切换到 B 分支，会把改变带过去。并且如果有冲突，会弹出下面的东西，点击 "Smart Checkout" 去解决冲突。
![[../../assets/Pasted image 20260829202538.png|345]]
但正常情况下，你肯定不希望把你的改变带过去，需要使用下面的命令：
```shell
# 切换分支时
git stash
# 切换回分支后
git stash pop

# 注意，我先在A分支，做了些改变，然后git stash；然后切到B分支，也做了些改变，git stash；然后切到A，git stash pop是上次B分支的东西
# 所以更好的习惯这样做
git stash push -m "TASK 123"
git stash list
# stash@{0}: On hmi_development: TASK 123
# stash@{1}: On hmi_development2: TASK 456
git stash apply stash@{0}
git stash drop stash@{0}
# 如果误删别的stash记录（执行完drop后，会打印出一个Hash值）
# git stash store -m "recovered dropped stash@{1}" f7fa65df047c0bd9f5383a8a243cfc3d270d1e83
```
2）如果已经 commit 了，是不会把你的改变带过去的，除非用 cherry-pick。
另外，我们经常需要把一些改变不仅提交到开发分支，还要提交到其它分支。通常我们会有一个开发分支 `hmi_development`，所有人日常开发的代码都要提到开发分支，包含最新的功能、Bug 修复；还有像发布分支 `hmi_development_release2`，这个分支只做稳定性修复，不再随意合并新功能，非 bug 类，或大的改动是不进入 R2 的；`hmi_development_hotfix6` 可能是当线上产品发现 Bug，拉出来的修复分支，这个分支只入 P2 bug，P3 bug 是不入的。
改完主分支提交后，想把改变应用到 R2 分支上，最方便的就是在 Gerrit 上点击 Cherry-Pick 按钮。有时会遇到冲突，则需用命令的方式去解。
```shell
# -n 不直接生成commit
git cherry-pick -n <commitID>

# 如果遇见冲突，还是改完<<，>>
git add <file>
git cherry-pick --continue
git cherry-pick --abort  # 放弃
```
3）- 如果想把 A 分支的多笔提交迁移到 B 分支，那就多次 `git cherry-pick`。这种时候还是不要用 `-n` 了，因为用了 `-n` 你再提交，这个提交是在你名下的。但这种时候，还是想保留原作者信息的，所以别用 `-n`。
#### 回退的方式
1）commit 后，想要回退到上一个提交，并且保留之前自己的改动。也就是说想要撤销提交，让你的之前的改变（绿色、蓝色的呢些）再次显现出来
```shell
git reset HEAD~1
# 最保险的还是--soft
# --soft：保留工作区、保留暂存区（git add）
# --mixed：保留工作区、清空暂存区（默认）
# --hard：清空工作区、清空暂存区

# git reset HEAD~2 就是连续回退两笔
```
2）拉下来有太多的冲突，或者自己输错了命令造成太多冲突，想直接让本地和远程最新代码完全一样。
```shell
git fetch
git reset --hard origin/hmi_development
```
3）通过查看命令，进行回退。
```shell
git reflog
# affc212d8 (HEAD -> task_1041298) HEAD@{0}: reset: moving to HEAD~1
# affc212d8 (HEAD -> task_1041298) HEAD@{0}: reset: moving to HEAD~1
# 2be7196ca HEAD@{1}: checkout: moving from Task_1093476 to task_1041298
# af594e4f3 (Task_1093476) HEAD@{2}: reset: moving to HEAD
# af594e4f3 (Task_1093476) HEAD@{3}: pull origin hmi_development: Fast-forward
# ...
# 这些信息在Termianl中只会显示一部分，需要按键盘的上下去看更多，按q退出
# git reflog是你所有的操作，如果只看某个分支，需要加上
git reflog hmi_development

git reset --hard <commitId>
```
4）已经和入代码后进行回退，这个命令和你手动删，然后 commit 差不多。不同的是 log 日志上 revert 语义更明显一点，并且 IDE 上可视化的颜色不一样。
```shell
# -n，只生成revert变更，不会提交
git revert -n <commit-hash>
```
5）想把本地的改动都删除
```shell
# 拿到改动文件路径
git status
# modified:   vehiclecontrol/src/vcupro/java/com/patac/hmi/vehiclecontrol/ui/FridgeOverlayManager.java
git restore vehiclecontrol/src/vcupro/java/com/patac/hmi/vehiclecontrol/ui/FridgeOverlayManager.java

# 恢复所有变更
git restore .
```
#### 如何重新触发编译
比如 Jenkins 抽风，Verified 失败，有时候重新触发一次就可以成功。这时候可以使用下面的命令，会重新做一次提交，但还是在同一个 Change 下，只是 commitId 不同。
```shell
git commit --amend
git push origin HEAD:refs/for/hmi_development_hotfix6
```
另外，代码没有 merge 的才能这样，有时侯 merge 进去了，后面 Jenkins 编译生成版本报错，那就没办法了，不是你的问题。
另外，如果真的遗漏提交了什么，可以下面这样：
```shell
# 更改完相应的文件
git add xxx
git commit --amend
git push origin HEAD:refs/for/hmi_development_hotfix6

# 使用amend时不要执行git pull，因为这可能导致本地分支不再是fast-forward状态
# 如果在amend之前执行了git pull，可能会拉取他人的提交，从而生成额外的merge commit，破坏原有的线性提交历史，将自己的提交和别人的提交和为一个，而且还提交不上去，必须git push --force
```
#### Gerrit 上遇到 Merge Conflict 怎么解决
现象是别人 +2 完后，“submit” 按钮变灰，出现 “Merge Conflict” 字样。导致这种情况的原因我现在发现两个：
1）和别人的代码提交冲突。
2）群上有时会看到无法入库的，SPM（软件项目经理 Software Project Manager）会说“有依赖的 change”、“另一笔依赖没有好”。这个依赖不是说依赖的 Lib 库这些，这里的意思其实是，当前这个 change（这笔提交）是在另一个尚未合并的 Change 基础上提交的，比如你刚改完代码，提了一笔（还没入库）；又在这笔的基础上继续写，又提了一笔，这样就会出现问题。此时可以直接使用 Gerrit 上的 rebase 按钮，基于已入库的最新主线提交，快捷解决。如果你这两笔有冲突只能等前一笔入了再说，或者 abondon 掉，重新一起提一笔。
![[../../assets/Pasted image 20260829203053.png|286]]
这时候不能使用 amend，因为 amend 只是更改你自己原来的提交，冲突是解决不了的，你和别人仍然是同时修改了同一个位置。
解决方法：
```shell
git fetch origin
# 把本地提交rebase到（变基到）最新主分支
git rebase origin/hmi_development
# 解决冲突文件
git add xxx
# 继续rebase
git rebase --continue
# ！！为了保证是同一个Change，push前先amend
git commit --amend
git push origin HEAD:refs/for/hmi_development

# git rebase --abort 撤销rebase
```
rebase 的本质就是把你的提交内容拿出来，基于最新的父提交再做一次提交。
```
# 你本地
A --- B --- C(你的改动)

# 远程
A --- B --- D(别人的改动)

git rebase origin/hmi_development
A --- B --- D --- C'
```
这里 C' 的 commit ID 会变，但是 Change-Id 不变，Gerrit 会生成新的 Patch Set，重新出发 Jenkins 编译，但是并不需要重新 review。
