# 在服务器上部署 Git

开始架设 Git 服务器前，需要先把现有仓库导出为裸仓库 — 即一个不包含当前工作目录的仓库。做法直截了当，克隆时用 --bare 选项即可。裸仓库的目录名一般以.git 结尾，像这样：

```
$ git clone --bare my_project my_project.git
Initialized empty Git repository in /opt/projects/my_project.git/
```

该命令的输出或许会让人有些不解。其实 clone 操作基本上相当于 git init 加 git fetch，所以这里出现的其实是git init 的输出，先由它建立一个空目录，而之后传输数据对象的操作并无任何输出，只是悄悄在幕后执行。现在my_project.git 目录中已经有了一份 Git 目录数据的副本。

整体上的效果大致相当于：

```
$ cp -Rf my_project/.git my_project.git
```

但在配置文件中有若干小改动，不过对用户来讲，使用方式都一样，不会有什么影响。它仅取出 Git 仓库的必要原始数据，存放在该目录中，而不会另外创建工作目录。

## 1.把裸仓库移到服务器上

有了裸仓库的副本后，剩下的就是把它放到服务器上并设定相关协议。假设一个域名为 git.example.com 的服务器已经架设好，并可以通过 SSH 访问，我们打算把所有 Git 仓库储存在/opt/git 目录下。只要把裸仓库复制过去：

```
$ scp -r my_project.git user@git.example.com:/opt/git
```

现在，所有对该服务器有 SSH 访问权限，并可读取 /opt/git 目录的用户都可以用下面的命令克隆该项目：

```
$ git clone user@git.example.com:/opt/git/my_project.git
```

如果某个 SSH 用户对 /opt/git/my_project.git 目录有写权限，那他就有推送权限。如果到该项目目录中运行 git init 命令，并加上 --shared 选项，那么 Git 会自动修改该仓库目录的组权限为可写（译注：实际上 --shared 可以指定其他行为，只是默认为将组权限改为可写并执行 g+sx，所以最后会得到 rws。）。

```
$ ssh user@git.example.com
$ cd /opt/git/my_project.git
$ git init --bare --shared
```

由此可见，根据现有的 Git 仓库创建一个裸仓库，然后把它放上你和同事都有 SSH 访问权的服务器是多么容易。现在已经可以开始在同一项目上密切合作了。

值得注意的是，这的的确确是架设一个少数人具有连接权的 Git 服务的全部 — 只要在服务器上加入可以用 SSH 登录的帐号，然后把裸仓库放在大家都有读写权限的地方。一切都准备停当，无需更多。

下面的几节中，你会了解如何扩展到更复杂的设定。这些内容包含如何避免为每一个用户建立一个账户，给仓库添加公共读取权限，架设网页界面，使用 Gitosis 工具等等。然而，只是和几个人在一个不公开的项目上合作的话，仅仅是一个 SSH 服务器和裸仓库就足够了，记住这点就可以了。

## 2.小型安装

如果设备较少或者你只想在小型开发团队里尝试 Git ，那么一切都很简单。架设 Git 服务最复杂的地方在于账户管理。如果需要仓库对特定的用户可读，而给另一部分用户读写权限，那么访问和许可的安排就比较困难。

## 3.SSH 连接

如果已经有了一个所有开发成员都可以用 SSH 访问的服务器，架设第一个服务器将变得异常简单，几乎什么都不用做（正如上节中介绍的那样）。如果需要对仓库进行更复杂的访问控制，只要使用服务器操作系统的本地文件访问许可机制就行了。

如果需要团队里的每个人都对仓库有写权限，又不能给每个人在服务器上建立账户，那么提供 SSH 连接就是唯一的选择了。我们假设用来共享仓库的服务器已经安装了 SSH 服务，而且你通过它访问服务器。
有好几个办法可以让团队的每个人都有访问权。第一个办法是给每个人建立一个账户，直截了当但略过繁琐。反复运行adduser 并给所有人设定临时密码可不是好玩的。

第二个办法是在主机上建立一个 git 账户，让每个需要写权限的人发送一个 SSH 公钥，然后将其加入 git 账户的~/.ssh/authorized_keys 文件。这样一来，所有人都将通过 git 账户访问主机。这丝毫不会影响提交的数据 — 访问主机用的身份不会影响提交对象的提交者信息。

另一个办法是让 SSH 服务器通过某个 LDAP 服务，或者其他已经设定好的集中授权机制，来进行授权。只要每个人都能获得主机的 shell 访问权，任何可用的 SSH 授权机制都能达到相同效果。

## 4.生成 SSH 公钥

大多数 Git 服务器都会选择使用 SSH 公钥来进行授权。系统中的每个用户都必须提供一个公钥用于授权，没有的话就要生成一个。生成公钥的过程在所有操作系统上都差不多。首先先确认一下是否已经有一个公钥了。SSH 公钥默认储存在账户的主目录下的~/.ssh 目录。进去看看：

```
$ cd ~/.ssh
$ ls
authorized_keys2  id_dsa       known_hosts
config            id_dsa.pub
```

关键是看有没有用 something 和 something.pub 来命名的一对文件，这个 something 通常就是 id_dsa 或 id_rsa。有 .pub后缀的文件就是公钥，另一个文件则是密钥。假如没有这些文件，或者干脆连.ssh 目录都没有，可以用 ssh-keygen 来创建。该程序在 Linux/Mac 系统上由 SSH 包提供，而在 Windows 上则包含在 MSysGit 包里：

```
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/schacon/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /Users/schacon/.ssh/id_rsa.
Your public key has been saved in /Users/schacon/.ssh/id_rsa.pub.
The key fingerprint is:
43:c5:5b:5f:b1:f1:50:43:ad:20:a6:92:6a:1f:9a:3a schacon@agadorlaptop.local
```

它先要求你确认保存公钥的位置（.ssh/id_rsa），然后它会让你重复一个密码两次，如果不想在使用公钥的时候输入密码，可以留空。

现在，所有做过这一步的用户都得把它们的公钥给你或者 Git 服务器的管理员（假设 SSH 服务被设定为使用公钥机制）。他们只需要复制 .pub 文件的内容然后发邮件给管理员。公钥的样子大致如下：

```
$ cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAklOUpkDHrfHY17SbrmTIpNLTGK9Tjom/BWDSU
GPl+nafzlHDTYW7hdI4yZ5ew18JH4JW9jbhUFrviQzM7xlELEVf4h9lFX5QVkbPppSwg0cda3
Pbv7kOdJ/MTyBlWXFCR+HAo3FXRitBqxiX1nKhXpHAZsMciLq8V6RjsNAQwdsdMFvSlVK/7XA
t3FaoJoAsncM1Q9x5+3V0Ww68/eIFmb1zuUFljQJKprrX88XypNDvjYNby6vw/Pb0rwert/En
mZ+AW4OZPnTPI89ZPmVMLuayrD2cE86Z/il8b+gw3r3+1nKatmIkjn2so1d01QraTlMqVSsbx
NrRFi9wrf+M7Q== schacon@agadorlaptop.local
```

关于在多个操作系统上设立相同 SSH 公钥的教程，可以查阅 GitHub 上有关 SSH 公钥的向导：http://github.com/guides/providing-your-ssh-key。

## 5.架设服务器

现在我们过一边服务器端架设 SSH 访问的流程。本例将使用 authorized_keys 方法来给用户授权。我们还将假定使用类似 Ubuntu 这样的标准 Linux 发行版。首先，创建一个名为 ‘git’ 的用户，并为其创建一个.ssh 目录。

```
$ sudo adduser git
$ su git
$ cd
$ mkdir .ssh
```

接下来，把开发者的 SSH 公钥添加到这个用户的 authorized_keys 文件中。假设你通过电邮收到了几个公钥并存到了临时文件里。重复一下，公钥大致看起来是这个样子：

```
$ cat /tmp/id_rsa.john.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCB007n/ww+ouN4gSLKssMxXnBOvf9LGt4L
ojG6rs6hPB09j9R/T17/x4lhJA0F3FR1rP6kYBRsWj2aThGw6HXLm9/5zytK6Ztg3RPKK+4k
Yjh6541NYsnEAZuXz0jTTyAUfrtU3Z5E003C4oxOj6H0rfIF1kKI9MAQLMdpGW1GYEIgS9Ez
Sdfd8AcCIicTDWbqLAcU4UpkaX8KyGlLwsNuuGztobF8m72ALC/nLF6JLtPofwFBlgc+myiv
O7TCUSBdLQlgMVOFq1I2uPWQOkOWQAHukEOmfjy2jctxSDBQ220ymjaNsHT4kgtZg2AYYgPq
dAv8JggJICUvax2T9va5 gsg-keypair
```

只要把它们逐个追加到 authorized_keys 文件尾部即可：

```
$ cat /tmp/id_rsa.john.pub >> ~/.ssh/authorized_keys
$ cat /tmp/id_rsa.josie.pub >> ~/.ssh/authorized_keys
$ cat /tmp/id_rsa.jessica.pub >> ~/.ssh/authorized_keys
```

现在可以用 --bare 选项运行 git init 来建立一个裸仓库，这会初始化一个不包含工作目录的仓库。

```
$ cd /opt/git
$ mkdir project.git
$ cd project.git
$ git --bare init
```

这时，Join，Josie 或者 Jessica 就可以把它加为远程仓库，推送一个分支，从而把第一个版本的项目文件上传到仓库里了。值得注意的是，每次添加一个新项目都需要通过 shell 登入主机并创建一个裸仓库目录。我们不妨以gitserver 作为 git 用户及项目仓库所在的主机名。如果在网络内部运行该主机，并在 DNS 中设定 gitserver 指向该主机，那么以下这些命令都是可用的：

在 John 的电脑上

```
$ cd myproject
$ git init
$ git add .
$ git commit -m 'initial commit'
$ git remote add origin git@gitserver:/opt/git/project.git
$ git push origin master
```

这样，其他人的克隆和推送也一样变得很简单：

```
$ git clone git@gitserver:/opt/git/project.git
$ vim README
$ git commit -am 'fix for the README file'
$ git push origin master
```

用这个方法可以很快捷地为少数几个开发者架设一个可读写的 Git 服务。

作为一个额外的防范措施，你可以用 Git 自带的 git-shell 工具限制 git 用户的活动范围。只要把它设为git 用户登入的 shell，那么该用户就无法使用普通的 bash 或者 csh 什么的 shell 程序。编辑 /etc/passwd 文件：

```
$ sudo vim /etc/passwd
```

在文件末尾，你应该能找到类似这样的行：

```
git:x:1000:1000::/home/git:/bin/sh
```

把 bin/sh 改为 /usr/bin/git-shell （或者用 which git-shell 查看它的实际安装路径）。该行修改后的样子如下：

```
git:x:1000:1000::/home/git:/usr/bin/git-shell
```

现在 git 用户只能用 SSH 连接来推送和获取 Git 仓库，而不能直接使用主机 shell。尝试普通 SSH 登录的话，会看到下面这样的拒绝信息：

```
$ ssh git@gitserver
fatal: What do you think I am? A shell?
Connection to gitserver closed.
```

## 6.公共访问

匿名的读取权限该怎么实现呢？也许除了内部私有的项目之外，你还需要托管一些开源项目。或者因为要用一些自动化的服务器来进行编译，或者有一些经常变化的服务器群组，而又不想整天生成新的 SSH 密钥 — 总之，你需要简单的匿名读取权限。

或许对小型的配置来说最简单的办法就是运行一个静态 web 服务，把它的根目录设定为 Git 仓库所在的位置，然后开启本章第一节提到的 post-update 挂钩。这里继续使用之前的例子。假设仓库处于/opt/git 目录，主机上运行着 Apache 服务。重申一下，任何 web 服务程序都可以达到相同效果；作为范例，我们将用一些基本的 Apache 设定来展示大体需要的步骤。

首先，开启挂钩：

```
$ cd project.git
$ mv hooks/post-update.sample hooks/post-update
$ chmod a+x hooks/post-update
```

如果用的是 Git 1.6 之前的版本，则可以省略 mv 命令 — Git 是从较晚的版本才开始在挂钩实例的结尾添加 .sample 后缀名的。

post-update 挂钩是做什么的呢？其内容大致如下：

```
$ cat .git/hooks/post-update 
#!/bin/sh
exec git-update-server-info
```

意思是当通过 SSH 向服务器推送时，Git 将运行这个 git-update-server-info 命令来更新匿名 HTTP 访问获取数据时所需要的文件。

接下来，在 Apache 配置文件中添加一个 VirtualHost 条目，把文档根目录设为 Git 项目所在的根目录。这里我们假定 DNS 服务已经配置好，会把对.gitserver 的请求发送到这台主机：

```   
  
    ServerName git.gitserver
    DocumentRoot /opt/git
    
               Order allow, deny
        allow from all
    
```

另外，需要把 /opt/git 目录的 Unix 用户组设定为 www-data ，这样 web 服务才可以读取仓库内容，因为运行 CGI 脚本的 Apache 实例进程默认就是以该用户的身份起来的：

```
$ chgrp -R www-data /opt/git
```

重启 Apache 之后，就可以通过项目的 URL 来克隆该目录下的仓库了。

```
$ git clone http://git.gitserver/project.git
```

这一招可以让你在几分钟内为相当数量的用户架设好基于 HTTP 的读取权限。另一个提供非授权访问的简单方法是开启一个 Git 守护进程，不过这将要求该进程作为后台进程常驻 — 接下来的这一节就要讨论这方面的细节。

### 1.GitWeb

现在我们的项目已经有了可读可写和只读的连接方式，不过如果能有一个简单的 web 界面访问就更好了。Git 自带一个叫做 GitWeb 的 CGI 脚本，运行效果可以到http://git.kernel.org 这样的站点体验下（见下图：基于网页的 GitWeb 用户界面）。

![Image of gitweb]		
(images/gitweb.png)
 
如果想看看自己项目的效果，不妨用 Git 自带的一个命令，可以使用类似 lighttpd 或 webrick 这样轻量级的服务器启动一个临时进程。如果是在 Linux 主机上，通常都预装了lighttpd ，可以到项目目录中键入 git instaweb 来启动。如果用的是 Mac ，Leopard 预装了 Ruby，所以webrick 应该是最好的选择。如果要用 lighttpd 以外的程序来启动 git instaweb，可以通过--httpd 选项指定：

```
$ git instaweb --httpd=webrick
[2009-02-21 10:02:21] INFO  WEBrick 1.3.1
[2009-02-21 10:02:21] INFO  ruby 1.8.6 (2008-03-03) [universal-darwin9.0]
```

这会在 1234 端口开启一个 HTTPD 服务，随之在浏览器中显示该页，十分简单。关闭服务时，只需在原来的命令后面加上--stop 选项就可以了：

```
$ git instaweb --httpd=webrick --stop
```

如果需要为团队或者某个开源项目长期运行 GitWeb，那么 CGI 脚本就要由正常的网页服务来运行。一些 Linux 发行版可以通过 apt 或yum 安装一个叫做 gitweb 的软件包，不妨首先尝试一下。我们将快速介绍一下手动安装 GitWeb 的流程。首先，你需要 Git 的源码，其中带有 GitWeb，并能生成定制的 CGI 脚本：

```
$ git clone git://git.kernel.org/pub/scm/git/git.git
$ cd git/
$ make GITWEB_PROJECTROOT="/opt/git" \
        prefix=/usr gitweb/gitweb.cgi
$ sudo cp -Rf gitweb /var/www/
```

注意，通过指定 GITWEB_PROJECTROOT 变量告诉编译命令 Git 仓库的位置。然后，设置 Apache 以 CGI 方式运行该脚本，添加一个 VirtualHost 配置：

```

    ServerName gitserver
    DocumentRoot /var/www/gitweb
    
               Options ExecCGI +FollowSymLinks +SymLinksIfOwnerMatch
        AllowOverride All
        order allow,deny
        Allow from all
        AddHandler cgi-script cgi
        DirectoryIndex gitweb.cgi
    
```

不难想象，GitWeb 可以使用任何兼容 CGI 的网页服务来运行；如果偏向使用其他 web 服务器，配置也不会很麻烦。现在，通过 http://gitserver 就可以在线访问仓库了，在http://git.server 上还可以通过 HTTP 克隆和获取仓库的内容。

### 2.Gitosis

把所有用户的公钥保存在 authorized_keys 文件的做法，只能凑和一阵子，当用户数量达到几百人的规模时，管理起来就会十分痛苦。每次改删用户都必须登录服务器不去说，这种做法还缺少必要的权限管理 — 每个人都对所有项目拥有完整的读写权限。

幸好我们还可以选择应用广泛的 Gitosis 项目。简单地说，Gitosis 就是一套用来管理 authorized_keys 文件和实现简单连接限制的脚本。有趣的是，用来添加用户和设定权限的并非通过网页程序，而只是管理一个特殊的 Git 仓库。你只需要在这个特殊仓库内做好相应的设定，然后推送到服务器上，Gitosis 就会随之改变运行策略，听起来就很酷，对吧？

Gitosis 的安装算不上傻瓜化，但也不算太难。用 Linux 服务器架设起来最简单 — 以下例子中，我们使用装有 Ubuntu 8.10 系统的服务器。

Gitosis 的工作依赖于某些 Python 工具，所以首先要安装 Python 的 setuptools 包，在 Ubuntu 上称为 python-setuptools：

```
$ apt-get install python-setuptools
```

接下来，从 Gitosis 项目主页克隆并安装：

```
$ git clone git://eagain.net/gitosis.git
$ cd gitosis
$ sudo python setup.py install
```

这会安装几个供 Gitosis 使用的工具。默认 Gitosis 会把 /home/git 作为存储所有 Git 仓库的根目录，这没什么不好，不过我们之前已经把项目仓库都放在/opt/git 里面了，所以为方便起见，我们可以做一个符号连接，直接划转过去，而不必重新配置：

```
$ ln -s /opt/git /home/git/repositories
```

Gitosis 将会帮我们管理用户公钥，所以先把当前控制文件改名备份，以便稍后重新添加，准备好让 Gitosis 自动管理authorized_keys 文件：

```
$ mv /home/git/.ssh/authorized_keys /home/git/.ssh/ak.bak
```

接下来，如果之前把 git 用户的登录 shell 改为 git-shell 命令的话，先恢复 ‘git’ 用户的登录 shell。改过之后，大家仍然无法通过该帐号登录（译注：因为authorized_keys 文件已经没有了。），不过不用担心，这会交给 Gitosis 来实现。所以现在先打开 /etc/passwd 文件，把这行：

```
git:x:1000:1000::/home/git:/usr/bin/git-shell
```

改回:

```
git:x:1000:1000::/home/git:/bin/sh
```

好了，现在可以初始化 Gitosis 了。你可以用自己的公钥执行 gitosis-init 命令，要是公钥不在服务器上，先临时复制一份：

```
$ sudo -H -u git gitosis-init < /tmp/id_dsa.pub
Initialized empty Git repository in /opt/git/gitosis-admin.git/
Reinitialized existing Git repository in /opt/git/gitosis-admin.git/
```

这样该公钥的拥有者就能修改用于配置 Gitosis 的那个特殊 Git 仓库了。接下来，需要手工对该仓库中的 post-update 脚本加上可执行权限：

```
$ sudo chmod 755 /opt/git/gitosis-admin.git/hooks/post-update
```

基本上就算是好了。如果设定过程没出什么差错，现在可以试一下用初始化 Gitosis 的公钥的拥有者身份 SSH 登录服务器，应该会看到类似下面这样：

```
$ ssh git@gitserver
PTY allocation request failed on channel 0
fatal: unrecognized command 'gitosis-serve schacon@quaternion'
  Connection to gitserver closed.
```

说明 Gitosis 认出了该用户的身份，但由于没有运行任何 Git 命令，所以它切断了连接。那么，现在运行一个实际的 Git 命令 — 克隆 Gitosis 的控制仓库：

在你本地计算机上

```
$ git clone git@gitserver:gitosis-admin.git
```

这会得到一个名为 gitosis-admin 的工作目录，主要由两部分组成：

```
$ cd gitosis-admin
$ find .
./gitosis.conf
./keydir
./keydir/scott.pub
```

gitosis.conf 文件是用来设置用户、仓库和权限的控制文件。keydir 目录则是保存所有具有访问权限用户公钥的地方— 每人一个。在keydir 里的文件名（比如上面的 scott.pub）应该跟你的不一样 — Gitosis 会自动从使用 gitosis-init 脚本导入的公钥尾部的描述中获取该名字。

看一下 gitosis.conf 文件的内容，它应该只包含与刚刚克隆的 gitosis-admin 相关的信息：

```
$ cat gitosis.conf 
[gitosis]

[group gitosis-admin]
writable = gitosis-admin
members = scott
```

它显示用户 scott — 初始化 Gitosis 公钥的拥有者 — 是唯一能管理 gitosis-admin 项目的人。

现在我们来添加一个新项目。为此我们要建立一个名为 mobile 的新段落，在其中罗列手机开发团队的开发者，以及他们拥有写权限的项目。由于 ‘scott’ 是系统中的唯一用户，我们把他设为唯一用户，并允许他读写名为iphone_project 的新项目：

```
[group mobile]
writable = iphone_project
members = scott
```

修改完之后，提交 gitosis-admin 里的改动，并推送到服务器使其生效：

```
$ git commit -am 'add iphone_project and mobile group'
[master]: created 8962da8: "changed name"
 1 files changed, 4 insertions(+), 0 deletions(-)
$ git push
Counting objects: 5, done.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 272 bytes, done.
Total 3 (delta 1), reused 0 (delta 0)
To git@gitserver:/opt/git/gitosis-admin.git
   fb27aec..8962da8  master -> master
```

在新工程 iphone_project 里首次推送数据到服务器前，得先设定该服务器地址为远程仓库。但你不用事先到服务器上手工创建该项目的裸仓库— Gitosis 会在第一次遇到推送时自动创建：

```
$ git remote add origin git@gitserver:iphone_project.git
$ git push origin master
Initialized empty Git repository in /opt/git/iphone_project.git/
Counting objects: 3, done.
Writing objects: 100% (3/3), 230 bytes, done.
Total 3 (delta 0), reused 0 (delta 0)
To git@gitserver:iphone_project.git
 * [new branch]      master -> master
```

请注意，这里不用指明完整路径（实际上，如果加上反而没用），只需要一个冒号加项目名字即可 — Gitosis 会自动帮你映射到实际位置。

要和朋友们在一个项目上协同工作，就得重新添加他们的公钥。不过这次不用在服务器上一个一个手工添加到~/.ssh/authorized_keys 文件末端，而只需管理keydir 目录中的公钥文件。文件的命名将决定在 gitosis.conf 中对用户的标识。现在我们为 John，Josie 和 Jessica 添加公钥：

```
$ cp /tmp/id_rsa.john.pub keydir/john.pub
$ cp /tmp/id_rsa.josie.pub keydir/josie.pub
$ cp /tmp/id_rsa.jessica.pub keydir/jessica.pub
```

然后把他们都加进 ‘mobile’ 团队，让他们对 iphone_project 具有读写权限：

```
[group mobile]
writable = iphone_project
members = scott john josie jessica
```

如果你提交并推送这个修改，四个用户将同时具有该项目的读写权限。

Gitosis 也具有简单的访问控制功能。如果想让 John 只有读权限，可以这样做：

```
[group mobile]
writable = iphone_project
members = scott josie jessica

[group mobile_ro]
readonly = iphone_project
members = john
```

现在 John 可以克隆和获取更新，但 Gitosis 不会允许他向项目推送任何内容。像这样的组可以随意创建，多少不限，每个都可以包含若干不同的用户和项目。甚至还可以指定某个组为成员之一（在组名前加上@ 前缀），自动继承该组的成员：

```
[group mobile_committers]
members = scott josie jessica

[group mobile]
writable  = iphone_project
members   = @mobile_committers

[group mobile_2]
writable  = another_iphone_project
members   = @mobile_committers john
```

如果遇到意外问题，试试看把 loglevel=DEBUG 加到 [gitosis] 的段落（译注：把日志设置为调试级别，记录更详细的运行信息。）。如果一不小心搞错了配置，失去了推送权限，也可以手工修改服务器上的/home/git/.gitosis.conf 文件 — Gitosis 实际是从该文件读取信息的。它在得到推送数据时，会把新的 gitosis.conf 存到该路径上。所以如果你手工编辑该文件的话，它会一直保持到下次向 gitosis-admin 推送新版本的配置内容为止。

### 3.Gitolite

Note: the latest copy of this section of the ProGit book is always available within thegitolite documentation. The author would also like to humbly state that, while this section is accurate, andcan (and often has) been used to install gitolite without reading any other documentation, it is of necessity not complete, and cannot completely replace the enormous amount of documentation that gitolite comes with.

Git has started to become very popular in corporate environments, which tend to have some additional requirements in terms of access control. Gitolite was originally created to help with those requirements, but it turns out that it’s equally useful in the open source world: the Fedora Project controls access to their package management repositories (over 10,000 of them!) using gitolite, and this is probably the largest gitolite installation anywhere too.
Gitolite allows you to specify permissions not just by repository, but also by branch or tag names within each repository. That is, you can specify that certain people (or groups of people) can only push certain “refs” (branches or tags) but not others.

Installing

Installing Gitolite is very easy, even if you don’t read the extensive documentation that comes with it. You need an account on a Unix server of some kind; various Linux flavours, and Solaris 10, have been tested. You do not need root access, assuming git, perl, and an openssh compatible ssh server are already installed. In the examples below, we will use thegitolite account on a host called gitserver.

Gitolite is somewhat unusual as far as “server” software goes – access is via ssh, and so every userid on the server is a potential “gitolite host”. As a result, there is a notion of “installing” the software itself, and then “setting up” a user as a “gitolite host”.

Gitolite has 4 methods of installation. People using Fedora or Debian systems can obtain an RPM or a DEB and install that. People with root access can install it manually. In these two methods, any user on the system can then become a “gitolite host”.

People without root access can install it within their own userids. And finally, gitolite can be installed by running a scripton the workstation, from a bash shell. (Even the bash that comes with msysgit will do, in case you’re wondering.)
We will describe this last method in this article; for the other methods please see the documentation.

You start by obtaining public key based access to your server, so that you can log in from your workstation to the server without getting a password prompt. The following method works on Linux; for other workstation OSs you may have to do this manually. We assume you already had a key pair generated using ssh-keygen.

```
$ ssh-copy-id -i ~/.ssh/id_rsa gitolite@gitserver
```

This will ask you for the password to the gitolite account, and then set up public key access. This isessential for the install script, so check to make sure you can run a command without getting a password prompt:

```
$ ssh gitolite@gitserver pwd
/home/gitolite
```

Next, you clone Gitolite from the project’s main site and run the “easy install” script (the third argument is your name as you would like it to appear in the resulting gitolite-admin repository):

```
$ git clone git://github.com/sitaramc/gitolite
$ cd gitolite/src
$ ./gl-easy-install -q gitolite gitserver sitaram
```

And you’re done! Gitolite has now been installed on the server, and you now have a brand new repository calledgitolite-admin in the home directory of your workstation. You administer your gitolite setup by making changes to this repository and pushing.

That last command does produce a fair amount of output, which might be interesting to read. Also, the first time you run this, a new keypair is created; you will have to choose a passphrase or hit enter for none. Why a second keypair is needed, and how it is used, is explained in the “ssh troubleshooting” document that comes with Gitolite. (Hey the documentation has to be good forsomething!)
Repos named gitolite-admin and testing are created on the server by default. If you wish to clone either of these locally (from an account that has SSH console access to the gitolite account viaauthorized_keys), type:

```
$ git clone gitolite:gitolite-admin
$ git clone gitolite:testing
To clone these same repos from any other account:
$ git clone gitolite@servername:gitolite-admin
$ git clone gitolite@servername:testing
```

Customising the Install

While the default, quick, install works for most people, there are some ways to customise the install if you need to. If you omit the-q argument, you get a “verbose” mode install – detailed information on what the install is doing at each step. The verbose mode also allows you to change certain server-side parameters, such as the location of the actual repositories, by editing an “rc” file that the server uses. This “rc” file is liberally commented so you should be able to make any changes you need quite easily, save it, and continue. This file also contains various settings that you can change to enable or disable some of gitolite’s advanced features.

Config File and Access Control Rules

Once the install is done, you switch to the gitolite-admin repository (placed in your HOME directory) and poke around to see what you got:

```
$ cd ~/gitolite-admin/
$ ls
conf/  keydir/
$ find conf keydir -type f
conf/gitolite.conf
keydir/sitaram.pub
$ cat conf/gitolite.conf
#gitolite conf
# please see conf/example.conf for details on syntax and features

repo gitolite-admin
    RW+                 = sitaram

repo testing
    RW+                 = @all
```

Notice that “sitaram” (the last argument in the gl-easy-install command you gave earlier) has read-write permissions on thegitolite-admin repository as well as a public key file of the same name.

The config file syntax for gitolite is liberally documented in conf/example.conf, so we’ll only mention some highlights here.

You can group users or repos for convenience. The group names are just like macros; when defining them, it doesn’t even matter whether they are projects or users; that distinction is only made when youuse the “macro”.

```
@oss_repos      = linux perl rakudo git gitolite
@secret_repos   = fenestra pear

@admins         = scott     # Adams, not Chacon, sorry :)
@interns        = ashok     # get the spelling right, Scott!
@engineers      = sitaram dilbert wally alice
@staff          = @admins @engineers @interns
```

You can control permissions at the “ref” level. In the following example, interns can only push the “int” branch. Engineers can push any branch whose name starts with “eng-“, and tags that start with “rc” followed by a digit. And the admins can do anything (including rewind) to any ref.

```
repo @oss_repos
    RW  int$                = @interns
    RW  eng-                = @engineers
    RW  refs/tags/rc[0-9]   = @engineers
    RW+                     = @admins
```

The expression after the RW or RW+ is a regular expression (regex) that the refname (ref) being pushed is matched against. So we call it a “refex”! Of course, a refex can be far more powerful than shown here, so don’t overdo it if you’re not comfortable with perl regexes.

Also, as you probably guessed, Gitolite prefixes refs/heads/ as a syntactic convenience if the refex does not begin withrefs/.

An important feature of the config file’s syntax is that all the rules for a repository need not be in one place. You can keep all the common stuff together, like the rules for alloss_repos shown above, then add specific rules for specific cases later on, like so:

```
repo gitolite
    RW+                     = sitaram
```

That rule will just get added to the ruleset for the gitolite repository.

At this point you might be wondering how the access control rules are actually applied, so let’s go over that briefly.

There are two levels of access control in gitolite. The first is at the repository level; if you have read (or write) access toany ref in the repository, then you have read (or write) access to the repository.

The second level, applicable only to “write” access, is by branch or tag within a repository. The username, the access being attempted (W or+), and the refname being updated are known. The access rules are checked in order of appearance in the config file, looking for a match for this combination (but remember that the refname is regex-matched, not merely string-matched). If a match is found, the push succeeds. A fallthrough results in access being denied.

Advanced Access Control with “deny” rules

So far, we’ve only seen permissions to be one or R, RW, orRW+. However, gitolite allows another permission: -, standing for “deny”. This gives you a lot more power, at the expense of some complexity, because now fallthrough is not theonly way for access to be denied, so the order of the rules now matters!
Let us say, in the situation above, we want engineers to be able to rewind any branchexcept master and integ. Here’s how to do that:

```
    RW  master integ    = @engineers
    -   master integ    = @engineers
    RW+                 = @engineers
```

Again, you simply follow the rules top down until you hit a match for your access mode, or a deny. Non-rewind push to master or integ is allowed by the first rule. A rewind push to those refs does not match the first rule, drops down to the second, and is therefore denied. Any push (rewind or non-rewind) to refs other than master or integ won’t match the first two rules anyway, and the third rule allows it.

Restricting pushes by files changed

In addition to restricting what branches a user can push changes to, you can also restrict what files they are allowed to touch. For example, perhaps the Makefile (or some other program) is really not supposed to be changed by just anyone, because a lot of things depend on it or would break if the changes are not done just right. You can tell gitolite:

```
repo foo
    RW                  =   @junior_devs @senior_devs

    RW  NAME/           =   @senior_devs
    -   NAME/Makefile   =   @junior_devs
    RW  NAME/           =   @junior_devs
```

This powerful feature is documented in conf/example.conf.

Personal Branches

Gitolite also has a feature called “personal branches” (or rather, “personal branch namespace”) that can be very useful in a corporate environment.
A lot of code exchange in the git world happens by “please pull” requests. In a corporate environment, however, unauthenticated access is a no-no, and a developer workstation cannot do authentication, so you have to push to the central server and ask someone to pull from there.
This would normally cause the same branch name clutter as in a centralised VCS, plus setting up permissions for this becomes a chore for the admin.
Gitolite lets you define a “personal” or “scratch” namespace prefix for each developer (for example,refs/personal/ /* ); see the “personal branches” section in doc/3-faq-tips-etc.mkd for details.

“Wildcard” repositories

Gitolite allows you to specify repositories with wildcards (actually perl regexes), like, for exampleassignments/s[0-9][0-9]/a[0-9][0-9], to pick a random example. This is avery powerful feature, which has to be enabled by setting$GL_WILDREPOS = 1; in the rc file. It allows you to assign a new permission mode (”C”) which allows users to create repositories based on such wild cards, automatically assigns ownership to the specific user who created it, allows him/her to hand out R and RW permissions to other users to collaborate, etc. This feature is documented indoc/4-wildcard-repositories.mkd.

Other Features

We’ll round off this discussion with a sampling of other features, all of which, and many more, are described in great detail in the “faqs, tips, etc” and other documents.

Logging: Gitolite logs all successful accesses. If you were somewhat relaxed about giving people rewind permissions (RW+) and some kid blew away “master”, the log file is a life saver, in terms of easily and quickly finding the SHA that got hosed.
Git outside normal PATH: One extremely useful convenience feature in gitolite is support for git installed outside the normal$PATH (this is more common than you think; some corporate environments or even some hosting providers refuse to install things system-wide and you end up putting them in your own directories). Normally, you are forced to make theclient-side git aware of this non-standard location of the git binaries in some way. With gitolite, just choose a verbose install and set$GIT_PATH in the “rc” files. No client-side changes are required after that :-)

Access rights reporting: Another convenient feature is what happens when you try and just ssh to the server. Gitolite shows you what repos you have access to, and what that access may be. Here’s an example:

```
    hello sitaram, the gitolite version here is v1.5.4-19-ga3397d4
    the gitolite config gives you the following access:
         R     anu-wsd
         R     entrans
         R  W  git-notes
         R  W  gitolite
         R  W  gitolite-admin
         R     indic_web_input
         R     shreelipi_converter
```

Delegation: For really large installations, you can delegate responsibility for groups of repositories to various people and have them manage those pieces independently. This reduces the load on the main admin, and makes him less of a bottleneck. This feature has its own documentation file in the doc/ directory.

Gitweb support: Gitolite supports gitweb in several ways. You can specify which repos are visible via gitweb. You can set the “owner” and “description” for gitweb from the gitolite config file. Gitweb has a mechanism for you to implement access control based on HTTP authentication, so you can make it use the “compiled” config file that gitolite produces, which means the same access control rules (for read access) apply for gitweb and gitolite.

Mirroring: Gitolite can help you maintain multiple mirrors, and switch between them easily if the primary server goes down.

## 7.Git 守护进程

对于提供公共的，非授权的只读访问，我们可以抛弃 HTTP 协议，改用 Git 自己的协议，这主要是出于性能和速度的考虑。Git 协议远比 HTTP 协议高效，因而访问速度也快，所以它能节省很多用户的时间。
重申一下，这一点只适用于非授权的只读访问。如果建在防火墙之外的服务器上，那么它所提供的服务应该只是那些公开的只读项目。如果是在防火墙之内的 服务器上，可用于支撑大量参与人员或自动系统（用于持续集成或编译的主机）只读访问的项目，这样可以省去逐一配置 SSH 公钥的麻烦。

但不管哪种情形，Git 协议的配置设定都很简单。基本上，只要以守护进程的形式运行该命令即可：

```
git daemon --reuseaddr --base-path=/opt/git/ /opt/git/
```

这里的 --reuseaddr 选项表示在重启服务前，不等之前的连接超时就立即重启。而 --base-path 选项则允许克隆项目时不必给出完整路径。最后面的路径告诉 Git 守护进程允许开放给用户访问的仓库目录。假如有防火墙，则需要为该主机的 9418 端口设置为允许通信。

以守护进程的形式运行该进程的方法有很多，但主要还得看用的是什么操作系统。在 Ubuntu 主机上，可以用 Upstart 脚本达成。编辑该文件：

```
/etc/event.d/local-git-daemon
加入以下内容：
start on startup
stop on shutdown
exec /usr/bin/git daemon \
    --user=git --group=git \
    --reuseaddr \
    --base-path=/opt/git/ \
    /opt/git/
respawn
```

出于安全考虑，强烈建议用一个对仓库只有读取权限的用户身份来运行该进程 — 只需要简单地新建一个名为 git-ro 的用户（译注：新建用户默认对仓库文件不具备写权限，但这取决于仓库目录的权限设定。务必确认git-ro 对仓库只能读不能写。），并用它的身份来启动进程。这里为了简化，后面我们还是用之前运行 Gitosis 的用户 ‘git’。

这样一来，当你重启计算机时，Git 进程也会自动启动。要是进程意外退出或者被杀掉，也会自行重启。在设置完成后，不重启计算机就启动该守护进程，可以运行：

```
initctl start local-git-daemon
```

而在其他操作系统上，可以用 xinetd，或者 sysvinit 系统的脚本，或者其他类似的脚本 — 只要能让那个命令变为守护进程并可监控。

接下来，我们必须告诉 Gitosis 哪些仓库允许通过 Git 协议进行匿名只读访问。如果每个仓库都设有各自的段落，可以分别指定是否允许 Git 进程开放给用户匿名读取。比如允许通过 Git 协议访问 iphone_project，可以把下面两行加到gitosis.conf 文件的末尾：

```
[repo iphone_project]
daemon = yes
```

在提交和推送完成后，运行中的 Git 守护进程就会响应来自 9418 端口对该项目的访问请求。

如果不考虑 Gitosis，单单起了 Git 守护进程的话，就必须到每一个允许匿名只读访问的仓库目录内，创建一个特殊名称的空文件作为标志：

```
$ cd /path/to/project.git
$ touch git-daemon-export-ok
```

该文件的存在，表明允许 Git 守护进程开放对该项目的匿名只读访问。

Gitosis 还能设定哪些项目允许放在 GitWeb 上显示。先打开 GitWeb 的配置文件 /etc/gitweb.conf，添加以下四行：

```
$projects_list = "/home/git/gitosis/projects.list";
$projectroot = "/home/git/repositories";
$export_ok = "git-daemon-export-ok";
@git_base_url_list = ('git://gitserver');
```

接下来，只要配置各个项目在 Gitosis 中的 gitweb 参数，便能达成是否允许 GitWeb 用户浏览该项目。比如，要让 iphone_project 项目在 GitWeb 里出现，把repo 的设定改成下面的样子：

```
[repo iphone_project]
daemon = yes
gitweb = yes
```

在提交并推送过之后，GitWeb 就会自动开始显示 iphone_project 项目的细节和历史。
