# 大项目的合并流程

Git 项目本身有四个长期分支：用于发布的 master 分支、用于合并基本稳定特性的 next 分支、用于合并仍需改进特性的pu分支（pu 是 proposed updates 的缩写），以及用于除错维护的 maint 分支（maint 取自 maintenance）。维护者可以按照之前介绍的方法，将贡献者的代码引入为不同的特性分支（如下图：管理复杂的并行贡献 所示），然后测试评估，看哪些特性能稳定工作，哪些还需改进。稳定的特性可以并入next 分支，然后再推送到公共仓库，以供其他人试用。

![Image of CVCS_24]		
(images/CVCS_24.png)

仍需改进的特性可以先并入 pu 分支。直到它们完全稳定后再并入 master。同时一并检查下 next 分支，将足够稳定的特性也并入 master。所以一般来说，master 始终是在快进，next 偶尔做下衍合，而pu 则是频繁衍合，如下图：将特性并入长期分支 所示：

![Image of CVCS_25]		
(images/CVCS_25.pg)

并入 master 后的特性分支，已经无需保留分支索引，放心删除好了。Git 项目还有一个 maint 分支，它是以最近一次发行版为基础分化而来的，用于维护除错补丁。所以克隆 Git 项目仓库后会得到这四个分支，通过检出不同分支可以了解各自进展，或是试用前沿特性，或是贡献代码。而维护者则通过管理这些分支，逐步有序地并入第三方贡献。

## 1.衍合与挑拣（cherry-pick）的流程
一些维护者更喜欢衍合或者挑拣贡献者的代码，而不是简单的合并，因为这样能够保持线性的提交历史。如果你完成了一个特性的开发，并决定将它引入到主干代码中，你可以转到那个特性分支然后执行衍合命令，好在你的主干分支上（也可能是develop分支之类的）重新提交这些修改。如果这些代码工作得很好，你就可以快进master分支，得到一个线性的提交历史。

另一个引入代码的方法是挑拣。挑拣类似于针对某次特定提交的衍合。它首先提取某次提交的补丁，然后试着应用在当前分支上。如果某个特性分支上有多个 commits，但你只想引入其中之一就可以使用这种方法。也可能仅仅是因为你喜欢用挑拣，讨厌衍合。假设你有一个类似下图：挑拣（cherry-pick）之前的历史 的工程。

![Image of CVCS_26]		
(images/CVCS_26.pg)
 
如果你希望拉取e43a6到你的主干分支，可以这样：

```
$ git cherry-pick e43a6fd3e94888d76779ad79fb568ed180e5fcdf
Finished one cherry-pick.
[master]: created a0a41a9: "More friendly message when locking the index fails."
 3 files changed, 17 insertions(+), 3 deletions(-)
```

这将会引入e43a6的代码，但是会得到不同的SHA-1值，因为应用日期不同。现在你的历史看起来像下图：挑拣（cherry-pick）之后的历史。

![Image of CVCS_27]		
(images/CVCS_27.pg)
 
现在，你可以删除这个特性分支并丢弃你不想引入的那些commit。

## 2.给发行版签名

你可以删除上次发布的版本并重新打标签，也可以像第二章所说的那样建立一个新的标签。如果你决定以维护者的身份给发行版签名，应该这样做：

```
$ git tag -s v1.5 -m 'my signed 1.5 tag'
You need a passphrase to unlock the secret key for
user: "Scott Chacon 
    
     "
1024-bit DSA key, ID F721C45A, created 2009-02-09
    
```

完成签名之后，如何分发PGP公钥（public key）是个问题。（译者注：分发公钥是为了验证标签）。还好，Git的设计者想到了解决办法：可以把key（既公钥）作为blob变量写入Git库，然后把它的内容直接写在标签里。gpg --list-keys命令可以显示出你所拥有的key：

```
$ gpg --list-keys
/Users/schacon/.gnupg/pubring.gpg
---------------------------------
pub   1024D/F721C45A 2009-02-09 [expires: 2010-02-09]
uid                  Scott Chacon 
    
sub   2048g/45D02282 2009-02-09 [expires: 2010-02-09]
    
```

然后，导出key的内容并经由管道符传递给git hash-object，之后钥匙会以blob类型写入Git中，最后返回这个blob量的SHA-1值：

```
$ gpg -a --export F721C45A | git hash-object -w --stdin
659ef797d181633c87ec71ac3f9ba29fe5775b92
```

现在你的Git已经包含了这个key的内容了，可以通过不同的SHA-1值指定不同的key来创建标签。

```
$ git tag -a maintainer-pgp-pub 659ef797d181633c87ec71ac3f9ba29fe5775b92
```

在运行git push --tags命令之后，maintainer-pgp-pub标签就会公布给所有人。如果有人想要校验标签，他可以使用如下命令导入你的key：

```
$ git show maintainer-pgp-pub | gpg --import
```

人们可以用这个key校验你签名的所有标签。另外，你也可以在标签信息里写入一个操作向导，用户只需要运行git show 查看标签信息，然后按照你的向导就能完成校验。

## 3.生成内部版本号

因为Git不会为每次提交自动附加类似’v123’的递增序列，所以如果你想要得到一个便于理解的提交号可以运行git describe命令。Git将会返回一个字符串，由三部分组成：最近一次标定的版本号，加上自那次标定之后的提交次数，再加上一段SHA-1值of the commit you’re describing：

```
$ git describe master
v1.6.2-rc1-20-g8c5b85c
```

这个字符串可以作为快照的名字，方便人们理解。如果你的Git是你自己下载源码然后编译安装的，你会发现git --version命令的输出和这个字符串差不多。如果在一个刚刚打完标签的提交上运行describe命令，只会得到这次标定的版本号，而没有后面两项信息。

git describe命令只适用于有标注的标签（通过-a或者-s选项创建的标签），所以发行版的标签都应该是带有标注的，以保证git describe能够正确的执行。你也可以把这个字符串作为checkout或者show命令的目标，因为他们最终都依赖于一个简短的SHA-1值，当然如果这个SHA-1值失效他们也跟着失效。最近Linux内核为了保证SHA-1值的唯一性，将位数由8位扩展到10位，这就导致扩展之前的git describe输出完全失效了。

## 4.准备发布

现在可以发布一个新的版本了。首先要将代码的压缩包归档，方便那些可怜的还没有使用Git的人们。可以使用git archive：

```
$ git archive master --prefix='project/' | gzip > `git describe master`.tar.gz
$ ls *.tar.gz
v1.6.2-rc1-20-g8c5b85c.tar.gz
```

这个压缩包解压出来的是一个文件夹，里面是你项目的最新代码快照。你也可以用类似的方法建立一个zip压缩包，在git archive加上--format=zip选项：

```
$ git archive master --prefix='project/' --format=zip > `git describe master`.zip
```

现在你有了一个tar.gz压缩包和一个zip压缩包，可以把他们上传到你网站上或者用e-mail发给别人。

## 5.制作简报

是时候通知邮件列表里的朋友们来检验你的成果了。使用git shortlog命令可以方便快捷的制作一份修改日志（changelog），告诉大家上次发布之后又增加了哪些特性和修复了哪些bug。实际上这个命令能够统计给定范围内的所有提交;假如你上一次发布的版本是v1.0.1，下面的命令将给出自从上次发布之后的所有提交的简介：

```
$ git shortlog --no-merges master --not v1.0.1
Chris Wanstrath (8):
      Add support for annotated tags to Grit::Tag
      Add packed-refs annotated tag support.
      Add Grit::Commit#to_patch
      Update version and History.txt
      Remove stray `puts`
      Make ls_tree ignore nils

Tom Preston-Werner (4):
      fix dates in history
      dynamic version method
      Version bump to 1.0.2
      Regenerated gemspec for version 1.0.2
```

这就是自从v1.0.1版本以来的所有提交的简介，内容按照作者分组，以便你能快速的发e-mail给他们。
