# 分支的管理

到目前为止，你已经学会了如何创建、合并和删除分支。除此之外，我们还需要学习如何管理分支，在日后的常规工作中会经常用到下面介绍的管理命令。

git branch 命令不仅仅能创建和删除分支，如果不加任何参数，它会给出当前所有分支的清单：

```
$ git branch
  iss53
* master
  testing
```

注意看 master 分支前的 * 字符：它表示当前所在的分支。也就是说，如果现在提交更新，master 分支将随着开发进度前移。若要查看各个分支最后一个提交对象的信息，运行git branch -v：

```
$ git branch -v
  iss53   93b412c fix javascript issue
* master  7a98805 Merge branch 'iss53'
  testing 782fd34 add scott to the author list in the readmes
```

要从该清单中筛选出你已经（或尚未）与当前分支合并的分支，可以用 --merge 和 --no-merged 选项（Git 1.5.6 以上版本）。比如用git branch --merge 查看哪些分支已被并入当前分支（译注：也就是说哪些分支是当前分支的直接上游。）：

```
$ git branch --merged
  iss53
* master
```

之前我们已经合并了 iss53，所以在这里会看到它。一般来说，列表中没有 * 的分支通常都可以用 git branch -d 来删掉。原因很简单，既然已经把它们所包含的工作整合到了其他分支，删掉也不会损失什么。

另外可以用 git branch --no-merged 查看尚未合并的工作：

```
$ git branch --no-merged
  testing
```

它会显示还未合并进来的分支。由于这些分支中还包含着尚未合并进来的工作成果，所以简单地用 git branch -d 删除该分支会提示错误，因为那样做会丢失数据：

```
$ git branch -d testing
error: The branch 'testing' is not an ancestor of your current HEAD.
If you are sure you want to delete it, run 'git branch -D testing'.
```

不过，如果你确实想要删除该分支上的改动，可以用大写的删除选项 -D 强制执行，就像上面提示信息中给出的那样。

## 1.利用分支进行开发的工作流程

现在我们已经学会了新建分支和合并分支，可以（或应该）用它来做点什么呢？在本节，我们会介绍一些利用分支进行开发的工作流程。而正是由于分支管理的便捷，才衍生出了这类典型的工作模式，你可以根据项目的实际情况选择一种用用看。

## 2.长期分支

由于 Git 使用简单的三方合并，所以就算在较长一段时间内，反复多次把某个分支合并到另一分支，也不是什么难事。也就是说，你可以同时拥有多个开放的分支，每个分支用于完成特定的任务，随着开发的推进，你可以随时把某个特性分支的成果并到其他分支中。

许多使用 Git 的开发者都喜欢用这种方式来开展工作，比如仅在 master 分支中保留完全稳定的代码，即已经发布或即将发布的代码。与此同时，他们还有一个名为develop 或 next 的平行分支，专门用于后续的开发，或仅用于稳定性测试 — 当然并不是说一定要绝对稳定，不过一旦进入某种稳定状态，便可以把它合并到master 里。这样，在确保这些已完成的特性分支（短期分支，比如之前的 iss53 分支）能够通过所有测试，并且不会引入更多错误之后，就可以并到主干分支中，等待下一次的发布。
本质上我们刚才谈论的，是随着提交对象不断右移的指针。稳定分支的指针总是在提交历史中落后一大截，而前沿分支总是比较靠前（见下图：稳定分支总是比较老旧）。

![Image of branch_18]		
(images/branch_18.png)

或者把它们想象成工作流水线，或许更好理解一些，经过测试的提交对象集合被遴选到更稳定的流水线（见下图：想象成流水线可能会容易点）。

![Image of branch_19]		
(images/branch_19.png)

你可以用这招维护不同层次的稳定性。某些大项目还会有个 proposed（建议）或 pu（proposed updates，建议更新）分支，它包含着那些可能还没有成熟到进入next 或 master 的内容。这么做的目的是拥有不同层次的稳定性：当这些分支进入到更稳定的水平时，再把它们合并到更高层分支中去。再次说明下，使用多个长期分支的做法并非必需，不过一般来说，对于特大型项目或特复杂的项目，这么做确实更容易管理。

## 3.特性分支

在任何规模的项目中都可以使用特性（Topic）分支。一个特性分支是指一个短期的，用来实现单一特性或与其相关工作的分支。可能你在以前的版本控 制系统里从未做过类似这样的事情，因为通常创建与合并分支消耗太大。然而在 Git 中，一天之内建立、使用、合并再删除多个分支是常见的事。

我们在上节的例子里已经见过这种用法了。我们创建了 iss53 和 hotfix 这两个特性分支，在提交了若干更新后，把它们合并到主干分支，然后删除。该技术允许你迅速且完全的进行语境切换 — 因为你的工作分散在不同的流水线里，每个分支里的改变都和它的目标特性相关，浏览代码之类的事情因而变得更简单了。你可以把作出的改变保持在特性分支中几 分钟，几天甚至几个月，等它们成熟以后再合并，而不用在乎它们建立的顺序或者进度。

现在我们来看一个实际的例子。请看下图：拥有多个特性分支的提交历史，由下往上，起先我们在 master 工作到 C1，然后开始一个新分支 iss91 尝试修复 91 号缺陷，提交到 C6 的时候，又冒出一个解决该问题的新办法，于是从之前 C4 的地方又分出一个分支iss91v2，干到 C8 的时候，又回到主干 master 中提交了 C9 和 C10，再回到 iss91v2 继续工作，提交 C11，接着，又冒出个不太确定的想法，从 master 的最新提交 C10 处开了个新的分支dumbidea 做些试验。

![Image of branch_20]		
(images/branch_20.png)

现在，假定两件事情：我们最终决定使用第二个解决方案，即 iss91v2 中的办法；另外，我们把 dumbidea 分支拿给同事们看了以后，发现它竟然是个天才之作。所以接下来，我们准备抛弃原来的iss91 分支（实际上会丢弃 C5 和 C6），直接在主干中并入另外两个分支。最终的提交历史将变成图 下图：合并了 dumbidea 和 iss91v2 后的分支历史 这样：

![Image of branch_21]		
(images/branch_21.png)

请务必牢记这些分支全部都是本地分支，这一点很重要。当你在使用分支及合并的时候，一切都是在你自己的 Git 仓库中进行的 — 完全不涉及与服务器的交互。

## 4.远程分支

远程分支（remote branch）是对远程仓库中的分支的索引。它们是一些无法移动的本地分支；只有在 Git 进行网络交互时才会更新。远程分支就像是书签，提醒着你上次连接远程仓库时上面各分支的位置。
我们用 (远程仓库名)/(分支名) 这样的形式表示远程分支。比如我们想看看上次同 origin 仓库通讯时master 的样子，就应该查看 origin/master 分支。如果你和同伴一起修复某个问题，但他们先推送了一个iss53 分支到远程仓库，虽然你可能也有一个本地的 iss53 分支，但指向服务器上最新更新的却应该是 origin/iss53 分支。

可能有点乱，我们不妨举例说明。假设你们团队有个地址为 git.ourcompany.com 的 Git 服务器。如果你从这里克隆，Git 会自动为你将此远程仓库命名为origin，并下载其中所有的数据，建立一个指向它的 master 分支的指针，在本地命名为origin/master，但你无法在本地更改其数据。接着，Git 建立一个属于你自己的本地master 分支，始于 origin 上 master分支相同的位置，你可以就此开始工作（见下图）：

下图表明了：一次 Git 克隆会建立你自己的本地分支 master 和远程分支 origin/master，它们都指向 origin/master 分支的最后一次提交

![Image of branch_22]		
(images/branch_22.png)

如果你在本地 master 分支做了些改动，与此同时，其他人向 git.ourcompany.com 推送了他们的更新，那么服务器上的master 分支就会向前推进，而于此同时，你在本地的提交历史正朝向不同方向发展。不过只要你不和服务器通讯，你的origin/master 指针仍然保持原位不会移动（见下图：在本地工作的同时有人向远程仓库推送内容会让提交历史开始分流）。

![Image of branch_23]		
(images/branch_23.png)

可以运行 git fetch origin 来同步远程服务器上的数据到本地。该命令首先找到 origin 是哪个服务器（本例为git.ourcompany.com），从上面获取你尚未拥有的数据，更新你本地的数据库，然后把 origin/master 的指针移到它最新的位置上（见下图：git fetch 命令会更新 remote 索引）。

![Image of branch_24]		
(images/branch_24.png)

为了演示拥有多个远程分支（在不同的远程服务器上）的项目是如何工作的，我们假设你还有另一个仅供你的敏捷开发小组使用的内部服务器 git.team1.ourcompany.com。可以用 Git 基础中提到的 git remote add 命令把它加为当前项目的远程分支之一。我们把它命名为 teamone，以便代替原始的 Git 地址（见下图：把另一个服务器加为远程仓库）。

![Image of branch_25]		
(images/branch_25.png)

现在你可以用 git fetch teamone 来获取小组服务器上你还没有的数据了。由于当前该服务器上的内容是你 origin 服务器上的子集，Git 不会下载任何数据，而只是简单地创建一个名为teamone/master 的分支，指向 teamone 服务器上 master 分支所在的提交对象31b8e（见下图：你在本地有了一个指向 teamone 服务器上 master 分支的索引）。

![Image of branch_26]		
(images/branch_26.png)

## 5.推送本地分支

要想和其他人分享某个本地分支，你需要把它推送到一个你拥有写权限的远程仓库。你的本地分支不会被自动同步到你引入的远程服务器上，除非你明确执行推送操作。换句话说，对于无意分享的分支，你尽管保留为私人分支好了，而只推送那些协同工作要用到的特性分支。

如果你有个叫 serverfix 的分支需要和他人一起开发，可以运行 git push (远程仓库名) (分支名)：

```
$ git push origin serverfix
Counting objects: 20, done.
Compressing objects: 100% (14/14), done.
Writing objects: 100% (15/15), 1.74 KiB, done.
Total 15 (delta 5), reused 0 (delta 0)
To git@github.com:schacon/simplegit.git
 * [new branch]      serverfix -> serverfix
```

这其实有点像条捷径。Git 自动把 serverfix 分支名扩展为 refs/heads/serverfix:refs/heads/serverfix，意为“取出我在本地的 serverfix 分支，推送到远程仓库的 serverfix 分支中去”。我们将在第九章进一步介绍refs/heads/ 部分的细节，不过一般使用的时候都可以省略它。也可以运行 git push origin serverfix:serferfix 来实现相同的效果，它的意思是“上传我本地的 serverfix 分支到远程仓库中去，仍旧称它为 serverfix 分支”。通过此语法，你可以把本地分支推送到某个命名不同的远程分支：若想把远程分支叫作awesomebranch，可以用 git push origin serverfix:awesomebranch 来推送数据。

接下来，当你的协作者再次从服务器上获取数据时，他们将得到一个新的远程分支 origin/serverfix：

```
$ git fetch origin
remote: Counting objects: 20, done.
remote: Compressing objects: 100% (14/14), done.
remote: Total 15 (delta 5), reused 0 (delta 0)
Unpacking objects: 100% (15/15), done.
From git@github.com:schacon/simplegit
 * [new branch]      serverfix    -> origin/serverfix
```

值得注意的是，在 fetch 操作下载好新的远程分支之后，你仍然无法在本地编辑该远程仓库中的分支。换句话说，在本例中，你不会有一个新的serverfix 分支，有的只是一个你无法移动的 origin/serverfix 指针。

如果要把该内容合并到当前分支，可以运行 git merge origin/serverfix。如果想要一份自己的 serverfix 来开发，可以在远程分支的基础上分化出一个新的分支来：

```
$ git checkout -b serverfix origin/serverfix
Branch serverfix set up to track remote branch refs/remotes/origin/serverfix.
Switched to a new branch "serverfix"
```

这会切换到新建的 serverfix 本地分支，其内容同远程分支 origin/serverfix 一致，这样你就可以在里面继续开发了。

## 6.跟踪远程分支

从远程分支 checkout 出来的本地分支，称为_跟踪分支(tracking branch)_。跟踪分支是一种和远程分支有直接联系的本地分支。在跟踪分支里输入git push，Git 会自行推断应该向哪个服务器的哪个分支推送数据。反过来，在这些分支里运行git pull 会获取所有远程索引，并把它们的数据都合并到本地分支中来。

在克隆仓库时，Git 通常会自动创建一个名为 master 的分支来跟踪 origin/master。这正是git push 和 git pull 一开始就能正常工作的原因。当然，你可以随心所欲地设定为其它跟踪分支，比如origin 上除了 master 之外的其它分支。刚才我们已经看到了这样的一个例子：git checkout -b [分支名] [远程名]/[分支名]。如果你有 1.6.2 以上版本的 Git，还可以用--track 选项简化：

```
$ git checkout --track origin/serverfix
Branch serverfix set up to track remote branch refs/remotes/origin/serverfix.
Switched to a new branch "serverfix"
```

要为本地分支设定不同于远程分支的名字，只需在前个版本的命令里换个名字：

```
$ git checkout -b sf origin/serverfix
Branch sf set up to track remote branch refs/remotes/origin/serverfix.
Switched to a new branch "sf"
```

现在你的本地分支 sf 会自动向 origin/serverfix 推送和抓取数据了。

## 7.删除远程分支

如果不再需要某个远程分支了，比如搞定了某个特性并把它合并进了远程的 master 分支（或任何其他存放稳定代码的地方），可以用这个非常无厘头的语法来删除它：git push [远程名] :[分支名]。如果想在服务器上删除serverfix 分支，运行下面的命令：

```
$ git push origin :serverfix
To git@github.com:schacon/simplegit.git
 - [deleted]         serverfix
```

咚！服务器上的分支没了。你最好特别留心这一页，因为你一定会用到那个命令，而且你很可能会忘掉它的语法。有种方便记忆这条命令的方法：记住我们不久前见过的 git push [远程名] [本地分支]:[远程分支] 语法，如果省略 [本地分支]，那就等于是在说“在这里提取空白然后把它变成[远程分支]”。
