# 分支的新建与切换

首先，我们假设你正在项目中愉快地工作，并且已经提交了几次更新（见下图：一个简短的提交历史）。

![Image of branch_10]		
(images/branch_10.png)

现在，你决定要修补问题追踪系统上的 #53 问题。顺带说明下，Git 并不同任何特定的问题追踪系统打交道。这里为了说明要解决的问题，才把新建的分支取名为 iss53。要新建并切换到该分支，运行git checkout 并加上 -b 参数：

```
$ git checkout -b iss53
Switched to a new branch "iss53"
```

这相当于执行下面这两条命令：

```
$ git branch iss53
$ git checkout iss53
```

下图示意该命令的执行结果：创建了一个新分支的指针。

![Image of branch_11]		
(images/branch_11.png)

接着你开始尝试修复问题，在提交了若干次更新后，iss53 分支的指针也会随着向前推进，因为它就是当前分支（换句话说，当前的 HEAD 指针正指向 iss53，见下图：分支随工作进展向前推进）：

```
$ vim index.html
$ git commit -a -m 'added a new footer [issue 53]'
```

![Image of branch_12]		
(images/branch_12.png)

现在你就接到了那个网站问题的紧急电话，需要马上修补。有了 Git ，我们就不需要同时发布这个补丁和 iss53 里作出的修改，也不需要在创建和发布该补丁到服务器之前花费大力气来复原这些修改。唯一需要的仅仅是切换回master 分支。

不过在此之前，留心你的暂存区或者工作目录里，那些还没有提交的修改，它会和你即将检出的分支产生冲突从而阻止 Git 为你切换分支。切换分支的时候最好保持一个清洁的工作区域。稍后会介绍几个绕过这种问题的办法（分别叫做 stashing 和 commit amending）。目前已经提交了所有的修改，所以接下来可以正常转换到master 分支：

```
$ git checkout master
Switched to branch "master"
```

此时工作目录中的内容和你在解决问题 #53 之前一模一样，你可以集中精力进行紧急修补。这一点值得牢记：Git 会把工作目录的内容恢复为检出某分支时它所指向的那个提交对象的快照。它会自动添加、删除和修改文件以确保目录的内容和你当时提交时完全一样。

接下来，你得进行紧急修补。我们创建一个紧急修补分支 hotfix 来开展工作，直到搞定（见下图：hotfix 分支是从 master 分支所在点分化出来的）：

```
$ git checkout -b 'hotfix'
Switched to a new branch "hotfix"
$ vim index.html
$ git commit -a -m 'fixed the broken email address'
[hotfix]: created 3a0874c: "fixed the broken email address"
 1 files changed, 0 insertions(+), 1 deletions(-)
```

![Image of branch_13]		
(images/branch_13.png)

有必要作些测试，确保修补是成功的，然后回到 master 分支并把它合并进来，然后发布到生产服务器。用 git merge 命令来进行合并：

```
$ git checkout master
$ git merge hotfix
Updating f42c576..3a0874c
Fast forward
 README |    1 -
 1 files changed, 0 insertions(+), 1 deletions(-)
```

请注意，合并时出现了“Fast forward”的提示。由于当前 master 分支所在的提交对象是要并入的 hotfix 分支的直接上游，Git 只需把master 分支指针直接右移。换句话说，如果顺着一个分支走下去可以到达另一个分支的话，那么 Git 在合并两者时，只会简单地把指针右移，因为这种单线的历史分支不存在任何需要解决的分歧，所以这种合并过程可以称为快进（Fast forward）。

现在最新的修改已经在当前 master 分支所指向的提交对象中了，可以部署到生产服务器上去了（见下图：合并之后，master 分支和 hotfix 分支指向同一位置）。

![Image of branch_14]		
(images/branch_14.png)

在那个超级重要的修补发布以后，你想要回到被打扰之前的工作。由于当前 hotfix 分支和 master 都指向相同的提交对象，所以hotfix 已经完成了历史使命，可以删掉了。使用 git branch 的 -d 选项执行删除操作：

```
$ git branch -d hotfix
Deleted branch hotfix (3a0874c).
```

现在回到之前未完成的 #53 问题修复分支上继续工作（图下图：iss53 分支可以不受影响继续推进）：

```
$ git checkout iss53
Switched to branch "iss53"
$ vim index.html
$ git commit -a -m 'finished the new footer [issue 53]'
[iss53]: created ad82d7a: "finished the new footer [issue 53]"
 1 files changed, 1 insertions(+), 0 deletions(-)
```

![Image of branch_15]		
(images/branch_15.png)

不用担心之前 hotfix 分支的修改内容尚未包含到 iss53 中来。如果确实需要纳入此次修补，可以用git merge master 把 master 分支合并到 iss53；或者等 iss53 完成之后，再将iss53 分支中的更新并入 master。
