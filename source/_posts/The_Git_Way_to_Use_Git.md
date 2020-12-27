---
title: 持续集成Git使用指南
date: 2019-01-01 00:00:00
tags:
---
发布系统Git使用指南

——the Git Way to Use Git
<!-- toc -->

### 背景

​	有文章曾归纳，Git是一套内容寻址文件系统，意思是，Git的核心是存储键值对^[1]^。显然，这样的形式不利于普通人类使用。

       通常情况下我们使用的Git命令，都被称作高级命令（例如pull、merge等），区别于底层的低级命令，两种命令分别对应于Git文档中出现Porcelain、Plumbing（第一次在文档见到这些词有没有很困惑！）。

       高级命令大都有易读参数与翔实输出，可以认为是由低级命令封装，方便人使用。那么低级命令存在的意义是？

       根据文档的描述^[2]^，低级命令的接口相较高级命令，更加稳定。低级指令本身就是为脚本使用而存在的，便于确切的输入参数并精准的识别输出。那么，在发布系统中合理使用低级命令替代高级命令，无疑将会使设计的逻辑更加准确的执行。

       以下，根据微店发布系统的实际场景和经验，从两个方面总结一下如何利用更恰当的Git命令，实现较完备的代码操作流程。

### 获取代码

       代码获取的大致逻辑分为四个层级：确定仓库、确定分支、确定Commit ID、确定目录

#### 仓库

       确定仓库地址之前，首先需要当前工作目录是否是一个Git仓库

```shell
chentairan@localhost ~/t/git-learn> git rev-parse --is-inside-work-tree
true
```

​	确认当前工作目录的仓库地址

```bash
chentairan@localhost ~/t/git-learn> git config --list|grep -F "remote.origin.url"
remote.origin.url=ssh://git@gitlab.xxx/chentairan/git-learn.git
```

​	如果需要直接设置仓库地址

```bash
chentairan@localhost ~/t/git-learn> git config remote.origin.url ssh://git@gitlab.xxx/chentairan/git-learn.git
```

#### 分支

​	获取指定远程分支

```shell
chentairan@localhost ~/t/git-learn> git fetch ssh://git@gitlab.xxx/chentairan/git-learn.git $BRANCH_NAME:refs/remotes/origin/$BRANCH_NAME
remote: Counting objects: 6, done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 6 (delta 0), reused 0 (delta 0)
Unpacking objects: 100% (6/6), done.
From ssh://gitlab.xxx/chentairan/git-learn
 * [new branch]      dev        -> origin/dev
```

​	获取分支的Commit ID

```shell
chentairan@localhost ~/t/git-learn> git rev-parse --verify refs/remotes/origin/$BRANCH_NAME
9e100c01201678286db5d266b8d342b0dd8a8e0d
```

​	检出提交

```shell
chentairan@localhost ~/t/git-learn> git checkout -f 9e100c01201678286db5d266b8d342b0dd8a8e0d
```

​	去掉多余文件

```shell
chentairan@localhost ~/t/git-learn> git clean -fdx
```

#### 多分支集成

​	在同时有多个RD一起开发的项目中，总会遇到有多个分支需要同时上线的情况（对应多个需求或者同一需求的不同工作部分）。通常的做法是从主干上拉出一个临时分支，将各开发分支的变更全部合并

```shell
chentairan@localhost ~/t/git-temp> git fetch origin master:refs/remotes/origin/master $BRANCH_DEV1:refs/remotes/origin/$BRANCH_DEV1 $BRANCH_DEV2:refs/remotes/origin/$BRANCH_DEV2
...
chentairan@localhost ~/t/git-temp> git checkout origin/master
chentairan@localhost ~/t/git-temp> git merge --no-edit --strategy=octopus origin/$BRANCH_DEV1 origin/$BRANCH_DEV2 ...
```

​	如果没有冲突，那么多分支的检出就完成了

#### Commit ID

​	如果发布时需要检出某指定提交，而不是某分支的最新提交（比如生产环境发布的代码，一定要是测试环境测试通过的代码版本），那么在fetch远程分支之后，直接检出对应提交

```shell
chentairan@localhost ~/t/git-temp> git checkout -f $COMMIT_ID
```

#### 目录

​	需要集成的项目代码可能不在项目根目录，而在项目下某一目录中，在使用`pushd $TARGET_DIRECTORY`之后，可能需要检查当前所在目录

```shell
chentairan@localhost ~/t/git-temp> pushd target_dir
chentairan@localhost ~/t/git-temp> git rev-parse --show-prefix
target_dir/
chentairan@localhost ~/t/git-temp> popd
chentairan@localhost ~/t/git-temp> git rev-parse --show-prefix

chentairan@localhost ~/t/git-temp> 
```

#### 本节回顾

`git rev-parse`用于查看参数，返回结果通常可以直接使用不需额外处理	

`git config`用于配置、查看参数，绕过了`git remote`命令可能存在的报错

`git fetch`用于拉取分支，并不改变当前工作区内容，相比`git pull`更加精确有效

`git checkout`用于检出提交，但对于缓存区文件（也许上次集成生成的问题）以及.gitignore中配置的文件不做处理

`git clean`用于清理工作区，仅留下指定提交下面关联的文件

`git merge --no-edit --strategy=octopus`用于合并多个分支，--no-edit参数使得CommitMessage不需要编辑，octopus是默认的Merge策略，在遇到复杂合并操作，需要人工解决时，拒绝合并

### 代码基线

​	基线管理简单说就两件事：确保开发代码包含代码主干的最新提交，确保代码主干有稳定版本可追溯。再说细一点：发布时检查代码是否合并了主干的最新提交，也即是最新的上线代码；线上发布完成之后，代码合并至主干，并打Tag。

#### 检查代码合并主干

​	检查是否合并了主干最新提交，通常有两种实现方式：一是在开发提交代码时，Git通过Hook配置检查；二是在发布系统这里，发布前做检查。方法一后续会写一篇文章专门讲Hook的能干的事情，本文主要介绍方法二。

​	查看代码主干位置

```shell
chentairan@localhost ~/t/git-temp> git ls-remote origin refs/heads/master
1297bb993557e8b58b37abc3dbb315402f95f349	refs/heads/master
```

​	查看开发代码是否包含此次提交历史

```shell
chentairan@localhost ~/t/git-temp> git rev-list $COMMIT_ID
......
f19d66e5a154e3b7415b36d55d6cdd9c3648261d
9e100c01201678286db5d266b8d342b0dd8a8e0d
1297bb993557e8b58b37abc3dbb315402f95f349
......
```

​	此处可以通过grep检查是否包含主干的最新提交

#### 向主干合并&Tag

​	由于开发的代码包含了主干的最新提交，那么向主干合并时，只是一个向前走的过程，一定不会有冲突，所以直接使用merge进行合并

```shell
...如第一部分拉取发布分支...
chentairan@localhost ~/t/git-temp> git checkout origin/master
chentairan@localhost ~/t/git-temp> git merge $COMMIT_ID --no-ff -m "$SOME_MESSAGE"
chentairan@localhost ~/t/git-temp> git tag $TAG_INFO -m "Create production tag"
chentairan@localhost ~/t/git-temp> git push origin HEAD:refs/heads/master --tags
```

####本节回顾	

`git ls-remote`用于查看远程代码仓库上主干分支的位置

`git rev-list`用于查看某次提交之前的历史

`git merge --no-ff` 用于合并代码。根据健康的Git工作流，所有的开发提交合并至主干时都是fast-forward过程。--no-ff参数能使fast-forward也产生一个新的Merge Commit（无此参数则不会），目的是保持开发分支的开发记录，保持整个代码开发历史的完整性

`git tag`用于新建Tag，保存线上稳定版本

`git push origin HEAD:refs/heads/master --tags`用于提交Merge后的代码及Tag。使用origin/master拉取代码，并使用HEAD:refs/heads/master方式提交，相比切换至本地master分支再更新、合并、提交，步骤更简单，并绕过本地master分支可能存在的需要处理的变更

### 提示&总结

​	发布系统的准确性直接影响线上稳定性，准确性的首要任务是确保异常产物无法上线

​	所有的命令均可以通过Shell的$?进行判错，有异常一定终止流程，人工介入，尽量避免使用--force参数

​	所有的命令以准确性为首要目标，脚本中所有的Git指令，能尽可能全面地指定参数，则尽量全面（曾经遇到过使用Git关键字作为分支名，导致的异常问题，Git偶尔会混淆在命令中该参数的具体含义指代。PS：不规范的名字可以在提交前通过Hook进行检查）

参考文献

[1][https://git-scm.com/book/zh/v1/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-Git-%E5%AF%B9%E8%B1%A1](https://git-scm.com/book/zh/v1/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-Git-%E5%AF%B9%E8%B1%A1)

[2] [http://man7.org/linux/man-pages/man1/git.1.html](http://man7.org/linux/man-pages/man1/git.1.html)

 
