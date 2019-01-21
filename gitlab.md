# GitLab


### 简介
一个基于GIT的源码托管解决方案
基于Ruby on rails开发
集成了Nginx PostgreSQL redis Sidekiq等组件
### 安装和部署

#### 资源
官网
https://about.gitlab.com/downloads

清华镜像
https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/
#### 系统要求
虚拟机centos7 64位/centos6

内存2G+

安装版本gitlab_ce_9.0.4以上
#### 安装依赖
```shell
sudo yum install curl policycoreutils openssh-server openssh-clients
sudo systemctl enable sshd
sudo systemctl start sshd
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
sudo firewall-cmd --permanent --add-service=http
sudo systemctl reload firewalld
```
#### 执行安装
```shell
rpm -ivh gitlab-ce-9.0.4-ce.0.el7.x86_64.rpm
修改配置文件 /etc/gitlab/gitlab.rb [external_url ] 为域名或者ip

gitlab-ctl reconfigure
```
访问: http://{{ip}}

输入:默认用户
root

至此: 安装gitlab安装完成.


### 常用命令及相关组件配置
#### 常用命令
*   restart
    Stop the services if they are running, then start them again.
*  service-list
    List all the services (enabled services appear with a *.)
*  start
    Start services if they are down, and restart them if they stop.
*  status
    Show the status of all the services.
*  stop
    Stop the services, and do not restart them.
*  tail
    Watch the service logs of all enabled services.

#### 相关组件常用配置文件目录
##### gitlab组件
nginx：静态Web服务器
gitlab-shell：用于处理Git命令和修改authorized keys列表
gitlab-workhorse:轻量级的反向代理服务器
logrotate：日志文件管理工具
postgresql：数据库
redis：缓存数据库
sidekiq：用于在后台执行队列任务（异步执行）
unicorn：GitLab Rails应用是托管在这个服务器上面的。

##### 配置目录
```shell
/var/opt/gitlab/git-data/repositories：代码库默认存储目录
/opt/gitlab：			应用代码和相应的依赖程序
/var/opt/gitlab：gitlab-ctl reconfigure命令编译后的应用数据和配置文件，不需要人为修改配置
/etc/gitlab：	配置文件目录
/var/log/gitlab：此目录下存放了gitlab各个组件产生的日志
/var/opt/gitlab/backups/：备份文件生成的目录[默认备份目录,可修改]
```
变更主配置文件后的操作
```shell
需要以下操作
1、gitlab-ctl reconfigure                  重置配置文件
2、gitlab-ctl show-config                   验证配置文件
3、gitlab-ctl restart                           重启gitlab服务
```
#### 创建对象
创建gourps
admin>add new gourp > add
创建用户
admin> add new users > add
创建项目
admin > new project > add 
授权项目用户
admin > groups//project >> add user to ..
#### ssh key管理
个人SSH KEY

Deploy KEY

创建SSH KEY
```shell
ssh-kengen -t dsa
cat ~/.ssh/id_rsa.pub
```
网页里将公钥导入用户SSHKEY

创建deploy key( 面向所有项目,JENKINS用)
将deploy key导入gitlab并在项目中允许

ssh key文件全局唯一
#### gitlab实践
在gitlab上创建一个库
>new project

用git上传文件
```shell
git clone xx@xxx.git
cd xxx
添加文件到 xxx

```
创建一个分支
```shell
git checkout -b [branch_name]

```
在分支上开发/推送
```shell
git add .
git commit -m "xxxx"
git push orgin [current_branch_name]
```
developer : web上发出merge request
manager:   Accept merge
### 权限管理及备份管理

#### issue管理(项目实际操作)
创建milestone
* project > new milestone > 

创建issue
* project > new issue > 添加开发人员> milestone 

创建分支
```shell
[root@docker02 JAVA]# git checkout -b dev1
切换到一个新分支 'dev1'
[root@docker02 JAVA]# git status
# 位于分支 dev1
无文件要提交，干净的工作区
[root@docker02 JAVA]# echo "<h1>welcome to gitlab</h1>" > index.html
[root@docker02 JAVA]# git add .
[root@docker02 JAVA]# git commit -m "add index"
[dev1 6a7b41d] add index
 1 file changed, 1 insertion(+)
 create mode 100644 index.html
[root@docker02 JAVA]# git push orgin dev1
fatal: 'orgin' does not appear to be a git repository
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
[root@docker02 JAVA]# git push origin dev1
Counting objects: 4, done.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 332 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote:
remote: To create a merge request for dev1, visit:
remote:   http://10.0.0.28/adm1/JAVA/merge_requests/new?merge_request%5Bsource_branch%5D=dev1
remote:
To git@10.0.0.28:adm1/JAVA.git
 * [new branch]      dev1 -> dev1
```
合并分支
developer : project >branch > merge request 
manager :  project > merge request > 对应request > 确认change> merge


>提示: git commit -m 可使用下面的两个关键字来直接命令行里自动fix 和close issue
Fix #issue_id
Close #issue_id

#### 备份管理及恢复
配置备份管理
vim /etc/gitlab/gitlab.rb
1.备份管理
配置文件中配置备份目录
gitlab_rails['backup_path'] = '/data/backup/gitlab'
[时间]秒,这里为7天
gitlab_rails['backup_keep_time'] = 604800

如果自定义备份目录需要赋予git权限
mkdir /data/backup/gitlab
chown -R git.git /data/backup/gitlab
定时任务Crontab中加入
0 2 * * * /usr/bin/gitlab-rake gitlab:backup:create   2&>1

策略建议：本地保留三到七天，在异地备份永久保存
```shell
gitlab-ctl reconfigure
gitlab-ctl restart
```
2.恢复
执行一次备份
```shell
gitlab-rake gitlab:backup:restore 

```
备份文件名为: 时间戳_日期_标_tar
时间戳可以命令行解析:
```shell
date -d @[时间戳]
```


#### 备份恢复实践
手工备份
/usr/bin/gitlab-rake gitlab:backup:create
记录系统状态
系统变更
```shell
停止数据写入服务
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
```
进行恢复
```shell
gitlab-rake gitlab:backup:restore BACKUP=[时间戳_日期]
两个yes确认.
gitlab-ctl restart
```
### gitlab邮件配置
gitlab_rails['time_zone'] = 'Asia/Shanghai'
gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['gitlab_email_from'] = 'guohongze@126.com'
gitlab_rails['gitlab_email_display_name'] = 'gitlab'
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.126.com"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_user_name'] = "rellata"
gitlab_rails['smtp_password'] = "your_password"
gitlab_rails['smtp_domain'] = "126.com"
gitlab_rails['smtp_authentication'] = "login"
##### gitlab慢的问题
#### 上线版本管理系统流程
* 搭建ntp服务器做时间同步
* git版本及客户端环境统一
* 搭建gitlab及授权服务器
* SourceTree客户端等