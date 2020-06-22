## Git操作失败并提示Another git process seems to be running in this......

> Git操作的过程中突然显示Another git process semms to be running in this repository, e.g. an editor opened by ‘git commit’. Please make sure all processes are terminated then try again. If it still fails, a git process remove the file manually to continue…

翻译:git被另外一个程序占用，重启机器也不能够解决.  
原因在于Git在使用过程中遭遇了奔溃，部分被上锁资源没有被释放导致的。  
解决方案：进入项目文件夹下的 .git文件中（显示隐藏文件夹或rm .git/index.lock）删除index.lock文件即可.


## 操作命令

- git help / git help -a
- git config 设置用户信息
- git config --global user.name "maoyz" //给自己起个用户名
- git config --global user.email  "abc@gmail.com" //填写自己的邮箱
- git config --unset --user.name "new_name" //重新设置用户名
- git config --list //列出信息

> 获取密钥  
    ssh-keygen -t rsa -C "abc@gmail.com" //填写email地址，然后一直“回车”ok;  
    打开本地..\.ssh\id_rsa.pub文件，將该密钥复制到github的setting/ssh中 ;  
连接GitHub服务器  
    ssh -T git@github.com;  
    目录下创建文件 touch  .html/.css/.js;  

- git init    初始化仓库
- git status  查看状态
- git add     添加文件到暂存区stage
- git commit  提交文件
- git commit -am "add XXX"  
     写入信息，然后按esc : wq  
           完成后提示信息：On branch master, nothing to commit,working tree clean  
     
- git log  查看日志
- git log --oneline
- git reflog  

- git mv 修改暂存区stage中文件名称
         mv和git mv区别
- git rm
> rm和git rm区别
git rm --cached <file>   删除stage中文件，保留工作区文件
git rm --f <file>     强制删除文件
12  git chenkout -- <file>      回滚


- 13  git revert HEAD     撤销某次操作，此次操作之前和之后的commit和history都会保留，并且把这次撤销作为一次最新的commit，
                        如果需要彻底回退，只需要将本次commit,会将本地和文本库中文件都删掉(HEAD相当于指针)
    git revert HEAD^^     上一个版本
    git revert HEAD～10
- 14  git reset HEAD^ --soft     缓存区存在,保留源码,只回退到commit 信息到某个版本.不涉及index的回退,如果还需要提交,直接commit即可
    git reset HEAD^ --mixed    会保留源码,只是将git commit和index 信息回退到了某个版本
    git reset HEAD^ --hard(不建议使用)
    git reset 版本号(建议使用)

git revert是用一次新的commit来回滚之前的commit，git reset是直接删除指定的commit
第一:上面我们说的如果你已经push到线上代码库, reset删除指定commit以后,你git push可能导致一大堆冲突.但是revert并不会.
第二:如果在日后现有分支和历史分支需要合并的时候,reset 恢复部分的代码依然会出现在历史分支里.但是revert方向提交的commit并不会出现在历史分支里.
第三:reset 是在正常的commit历史中,删除了指定的commit,这时HEAD是向后移动了,而revert是在正常的commit历史中再commit一次,只不过是反向提交,他的HEAD是一直向前的.

15  分支
    查看分支   git branch
    创建分支   git branch tag1  
    分支重命名  git branch -m old-tag1 new-tag  
    使用分支   git checkout tag1  -> 
              git checkout -b bug__001/feather__1
    合并分支   git checkout master
              git merge tag1  ->Fast forward
    合并冲突: (重要)      
                   git merge tag1 --no-ff -m"message"     (no fast forward)
    刪除分支   git branch -d tag1
    封存分支   git stash
               git stash list
    解除封存   git stash pop
               git stash apply stash@{n}

git diff (master..tag1/tag2)     (多用)



## 同时同步至github和gitee(码云)

### 方法一：命令

1. 先删除已关联的名为origin的远程库：

```shell
git remote rm origin
```

2. 关联远程库

```shell
# 关联GitHub
git remote add github git@github.com:chloneda/demo.git
# 关联码云
git remote add gitee git@gitee.com:chloneda/demo.git
```



### 方法二：配置文件

修改.git文件夹内的config文件：

```shell
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
[remote "origin"]
    url = git@github.com:chloneda/demo.git
    fetch = +refs/heads/*:refs/remotes/github/*
[branch "master"]
    remote = origin
    merge = refs/heads/master
```

将[remote "origin"]内容复制，修改origin名称，内容如下:

```shell
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
[remote "github"]
    url = git@github.com:chloneda/demo.git
    fetch = +refs/heads/*:refs/remotes/github/*
[remote "gitee"]
    url = git@gitee.com:chloneda/demo.git
    fetch = +refs/heads/*:refs/remotes/gitee/*
[branch "master"]
    remote = origin
    merge = refs/heads/master
```



### 提交

```shell
git push github master
git push gitee master
```



### 更新

```shell
# 从github拉取更新
git pull github

# 从gitee拉取更新
git pull gitee
```



##  git 中文文件名乱码

git默认中文文件名是\xxx\xxx 等八进制形式，对0x80以上的字符进行quote（引用），正常显示中文：

```shell
git config --global core.quotepath false
```



