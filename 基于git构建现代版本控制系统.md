# 2.分布式版本控制利器git

## git 基础
### git简介
* linus 用C编写
* 05年诞生
* 分布式版本管理系统
* 速度快,适合大规模,跨地区多人协作

### git生态
* git 分布式版本管理系统
* gitlab git私库解决方案
* github git公有库解决方案

### git安装
* Centos 
yum install git
* Ubuntu 
apt-get install git

* Windows安装git bash
* Liunx编译安装
注意不要使用git1.8以下版本

### git命令
设置与配置 git config 
帮助命令git help 

初始化
git init
git config --global user.name "guohongze"
git config --global user.email guohongze@126.com

获取
git clone http://xxx.git

git add 		加入暂存（索引区）
git status 		查看状态
git status -s	状态概览
git diff		尚未暂存的文件
git diff --staged    暂存区文件
git commit 	提交更新
git reset 		回滚
git rm 		从版本库中移除
git rm --cached README 从暂存区中移除
git mv 		相当于mv git rm git add三个命令

### 四个区域
远程仓库-本地仓库-暂存区域-工作目录
![441e76ec066adde072fadc76b1363d43.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2224)


## 分支管理
![1264fc6ffe2c2e482b326a447738c2da.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2225)
```shell
git branch testing
```
会在当前所在的提交对象上创建一个指针
![7eb004581ca4a14f59b944de8f59d18d.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2226)
Head 指向当前所在的分支
![5601bbd6e460c85b4b97d30da6be7d68.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2227)
```shell
git checkout testing
```
HEAD指向testing分支
![7006f91243494c6ab3705f5848391c5f.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2228)
### 分支命令
git branch 
git branch –v
git branch –merged
git branch --no-merged
git branch -d testing
git checkout 
git merge 
git log 
git stash 
git tag
![503cddac985da32bb40e2f17437d983b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2229)

## git高级管理
### git checkout 两个功能
git checkout 命令：用于切换分支。
git checkout -- file.ext 撤销对文件的修改

Checkout一个文件和带文件路径git reset 非常像，除了它更改的是工作目录而不是缓存区。不像提交层面的checkout命令，它不会移动HEAD引用，也就是你不会切换到别的分支上去。

如果你缓存并且提交了checkout的文件，它具备将某个文件回撤到之前版本的效果。注意它撤销了这个文件后面所有的更改，而git revert 命令只撤销某个特定提交的更改。
### git reset

--soft 	缓存区和工作目录都不会被改变

--mixed	默认选项。缓存区和你指定的提交同步，但工作目录不受影响

--hard	缓存区和工作目录都同步到你指定的提交
![94e22e79867c1c853d4e51dfb98ea590.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2231)

![484af25d07cbc19d71529a9eb65e6f60.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2234)
reset 参数影响
![b2956f57dcd0857a4bb44c6522db16e9.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2233)
**文件层操作**
当检测到文件路径时，git reset 将缓存区同步到你指定的那个提交。比如，下面这个命令会将倒数第二个提交中的foo.py加入到缓存区中，供下一个提交使用。

git reset HEAD~2 foo.py

运行git reset HEAD foo.py 会将当前的foo.py从缓存区中移除出去，而不会影响工作目录中对foo.py的更改。

--soft、--mixed和--hard对文件层面的git reset毫无作用，因为缓存区中的文件一定会变化，而工作目录中的文件一定不变。

![a9dba08f10a90b395f314b072d540e4a.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2235)

使用场景
命令	作用域	常用情景
git reset	提交层面	版本回滚，在私有分支上舍弃一些没有提交的更改
git reset	文件层面	将文件从缓存区中移除
git checkout	提交层面	切换分支或查看旧版本
git checkout	文件层面	舍弃工作目录中的更改
git revert	提交层面	在公共分支上回滚更改
git revert	文件层面	（没有）

### reflog
git reflog 命令分析你所有分支的头指针的日志来查找出你在重写历史上可能丢失的提交。
## 远程管理
![4755a7555e0eb974791bc8ef2de2ddb8.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p2230)

### 远程管理命令

* git clone https://github.com/xxxx/xxx.git
* git pull
* git fetch
* git push origin master
* git remote 
* git remote –v
* git remote add xxx http://xxx
* git remote show origin
* git remote rename pb paul
* git tag -a v1.0 -m ‘abc’

### 标签管理

* git tag -a v1.4 -m 'my version 1.4‘
* git show v1.4
* git tag -a v1.2 9fceb02			对历史打标签
* git push origin v1.5			将标签推向远程
* git push origin --tags			推送多个标签
* git checkout -b version2 v2.0.0	检出标签