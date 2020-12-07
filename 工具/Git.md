### 什么是Git？

Git是一种版本控制系统（VCS，Version control system），记录一个或者若干文件内容变化，以便将来查阅特定版本的修订情况



### Git commit graph

Git追踪时间变化的文件，执行commit命令记录了某个特定时间点的文件的状态

### 3 areas of a git repo

在git中，有3个逻辑区用于处理文件

- 工作树（Working Tree）
- 暂存区（Staging area），也称索引（Index）
- 历史记录（Git history）



### 什么是.git目录?

.git目录包含一个对象数据库和构成我们存储库的元数据

### 什么是.gitignore文件？

有些文件不需要来追踪，可以在.gitignore文件设置对应的文件路径，来忽略该文件的追踪

### 什么是Branch，如何实现？

 分支允许我们并行处理同一文件的不同版本，在一个分支上进行修改，可以独立于其他分支上的工作，最后可以决定是否合并修改到其他分支中

每次commit后，会生成40位的十六进制sha-1哈希，默认情况下Git创建了我们的第一个分支，master，它实际就是一个指向sha-1哈希值的一个指针，它总是跟踪着并且指向我们工作过程的最前端

HEAD是一个指向分支的指针，也被称为符号指针（symbolic poiner）

### Git object

blob ，文件内容

tree，目录结构、文件权限、文件名

commit，上一个commit、对应快照、作者、提交信息

hrefs，不是Git Object，是一个指针，Head，branch，tag

### 什么要把文件的权限和文件名存储在Tree object而不是Blob object呢？

如果要放在Blob object中，在修改文件名的时候，都需要新建一个Blob object出来，而在Tree object中只需要新建一个Tree object，Blob object中通常存储的是实际的内容，所占用的空间比较大

### 每次commit，Git存储的是全新的文件快照还是存储文件的变更部分？

全新的文件快照，这样可以使得每次取一个东西出来在O（1）的时间复杂度，直接找到tree object，把里面的东西拿出来

如果储存的是变更的部分，那么就需要遍历tree object里的文件快照

### Git如何保证历史记录不被篡改？

Git和区块链的数据结构非常相似，两者都基于哈希树和分布式，历史记录的某一个节点的哈希值发生改变，那么从它往上所有的哈希值都会发生改变

## Git命令

### git config --list

查看配置列表

### git status

告诉我们工作树和暂存区中的状态

### git add

将一个未追踪的文件转换为追踪文件

### git committed

git commit -m " "

将暂存区中的文件进行提交

### git log

告诉我们有关提交图的信息

### git diff

查看工作树和暂存区域中追踪文件的区别

### git diff --staged

查看暂存区和我们最近commit之间的区别

### git rm

删除工作树以及暂存区

### git checkout

使用暂存区域更早的版本来替换掉当前工作树中的版本

### git checkout `<branch name>`

切换分支

### git reset HEAD s1

从最新提交中恢复s1

### 查看分支图

git log --graph --decorate --oneline --simplify-by-decoration --all

### git branch `<branch name>`

分支在HEAD指针指向的位置进行实例化

### git merge `<branch name>`

合并分支

#### fast forward 

快速合并，将master分支移到`<branch name>`所指定的分支，需要保证有一条直接的路径

#### 3-way merge

三向合并，当合并的分支到matser没有一条直接的路径，此时不能直接移动master指针，因为它所在的指针出可能有别的分支，若移动了就获取不到这个分支的修改了



#### merge conflicts

合并冲突，当尝试合并更改了同一个文件相同行的分支时，会发生合并冲突，发送冲突时会在对应的文件中添加`>>>>>>> ==========`标记，在修改之后，再次进行add，commit即可



### git stash

当有未完成的工作时，可以暂时保存，获得干净的工作空间

 git stash save 

添加额外的说明

git stash apply 恢复

### git remote

显示远程仓库 

### git fetch

通过网络获取远程内容

### git pull

将git fetch 和 git merge组合为一个命令

### git push



### git reflog 时光机

可以查看对应操作的哈希值，找到对应的哈希值，就可以执行git reset撤销删除等操作

### git bisect

二分