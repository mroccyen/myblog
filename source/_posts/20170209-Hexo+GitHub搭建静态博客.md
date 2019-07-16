---
title: Hexo+GitHub搭建静态博客
date: 2017-02-09 10:01:51
tags: 
- hexo
categories: 
- [hexo]
---

### 开始

在安装[hexo](https://hexo.io/zh-cn/)之前，必须确认你已经安装了[Node.js](http://nodejs.org/)和[Git](http://git-scm.com/)，并且注册了一个[GitHub](https://github.com/)账号。
#### 1.创建Github仓库
##### 1) 仓库名为xxx.github.io
创建一个以"用户名.github.io"命名的仓库，如我的用户名为mroccyen,那我的仓库名为：[mroccyen.github.io](https://github.com/mroccyen/mroccyen.github.io)，仓库默认有master分支，用于托管生成的静态文件，再新建一个develop(名字自定)分支，用于托管后台文件，方便以后换电脑时后台文件不会丢失。
##### 2) 仓库名为其他名称，如myblog
如果是以其他名字命名的，如[myblog](https://github.com/mroccyen/myblog)，则需要建立一个gh-pages分支用于托管生成的静态文件，必须是gh-pages分支。后台文件可以托管在master分支上，也可以放在其他的分支。
#### 2.配置Git
设置Git的用户名和邮件地址（邮箱就是你注册Github时候的邮箱），打开Git Bash,键入：
``` bash
$ git config --global user.name "username"
$ git config --global user.email "email@example.com"
```
<!--more-->

#### 3.本地Git与Github建立联系

这里介绍SSH的配置，先检查电脑是否已经有SSH
``` bash
$ ls -al ~/.ssh
```
如果不存在就没有关系，如果存在的话，直接删除.ssh文件夹里面所有文件。
输入以下指令后，一路回车就好：
``` bash
$ ssh-keygen -t rsa -C "emailt@example.com"
```
然后键入以下指令：
``` bash
$ ssh-agent -s
$ ssh-add ~/.ssh/id_rsa
```
如果出现这个错误:`Could not open a connection to your authentication agent`，则先执行如下命令即可：
``` bash
$ ssh-agent bash
```
再重新输入指令：
``` bash
$ ssh-add ~/.ssh/id_rsa
```
到了这一步，就可以添加SSH key到你的Github账户了。键入以下指令，拷贝Key（先拷贝了，等一下可以直接粘贴）：
``` bash
$ clip < ~/.ssh/id_rsa.pub
```
在github上点击你的头像-->Your profile-->Edit profile-->SSH and GPG keys-->New SSH key
Title自己随便取，然后这个Key就是刚刚拷贝的，你直接粘贴就好（也可以文本打开id_rsa.pub复制其内容），最后Add SSH key。
最后还是测试一下吧，键入以下命令：
``` bash
$ ssh -T git@github.com
```
你可能会看到有警告，没事，输入“yes”就好。
#### 4.初始化hexo文件夹
如果建立的是xxx.github.io仓库，到GitHub的xxx.github.io仓库下，如果建立的是其他名字的仓库，则到其他名字仓库下，我这里是创建myblog，然后点击Clone or download，复制里面的HTTPS地址。
在你本地电脑的文件夹下，右键Git Bash Here: 键入git clone <刚复制的地址>。
``` bash
$ git clone https://github.com/mroccyen/myblog.git
$ mkdir myblog
```

### Hexo安装配置

#### 1.Hexo初始化
进入myblog文件夹
``` bash
$ cd myblog
```
接下来只需要使用 npm 即可完成 Hexo 的安装，如果在当前的文件夹下始终安装失败，到C盘里去安装，会成功。
``` bash
$ npm install -g hexo-cli
```

安装 Hexo 完成后，请执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件:
``` bash
$ hexo init
$ npm install
```
接下来也可以本地预览博客，执行下列命令,然后到浏览器输入localhost:4000看看。
``` bash
$ hexo generate
$ hexo server
```
输入Ctrl+C停止服务。
#### 2.Hexo配置
用编辑器打开 myblog/ 下的配置文件_config.yml找到：
``` bash
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: 
  repository:
```
到GitHub的myblog仓库下，点击Clone or download,复制里面的HTTPS地址到repository:，添加branch: gh-pages
``` bash
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: https://github.com/mroccyen/myblog.git
  branch: gh-pages
```
#### 3.完成部署
最后一步，快要成功了，键入指令：
``` bash
$ npm install hexo-deployer-git --save
$ hexo generate
$ hexo deploy
```
输入弹出框的用户名与密码(首次使用git会弹出)。
OK，我们的博客就已经完全搭建起来了，在浏览器输入（当然，是你的Repository名），例如我的：[myblog](https://github.com/MrocCyen/myblog/tree/gh-pages)
每次修改本地文件后，需要键入hexo generate才能保存，再键入hexo deploy上传文件。成功之后命令行最后两句大概是这样：
``` bash
    To https://github.com/mroccyen/myblog.git
       7f3b50a..128a10d  HEAD -> gh-pages
    INFO  Deploy done: git
```
这样生成的静态文件就上传到github的[myblog](https://github.com/MrocCyen/myblog/tree/gh-pages)仓库中的gh-pages分支中了。最后再把myblog文件夹中的所有文件上传到github中的myblog仓库中的master分之中。

### 日常操作

#### 1.写文章
执行new命令，生成指定名称的文章至 myblog\source\_posts\文章标题.md 。 
``` bash
$ hexo new [layout] "文章标题" #新建文章
```
然后用编辑器打开“文章标题.md”按照[Markdown语法](http://www.appinn.com/markdown/)书写文章。
 其中layout是可选参数，默认值为post。到 scaffolds 目录下查看现有的layout。当然你可以添加自己的layout，
 同时你也可以编辑现有的layout，比如post的layout默认是 myblog\scaffolds\post.md
``` bash
title: { { title } }
date: { { date } }
tags:
- tag1
- tag2
---
```
我想添加categories，以免每次手工输入，只需要修改这个文件添加如下行：
``` bash
title: { { title } }
date: { { date } }
categories:
- categories1
- categories2
tags:
---
```
文件标题也是md文件的名字，同时也出现在你文章的URL中，postName如果包含空格，必须用"将其包围。
`请注意，大括号与大括号之间我多加了个空格，否则会被转义，不能正常显示；所有文件"："后面都必须有个空格，不然会报错。`
#### 2.提交
每次在本地对博客进行修改后，先执行下列命令提交网站相关的文件。
``` bash
$ git add .
$ git commit -m "你的描述"
$ git push origin master
```
然后才执行hexo generate发布网站到gh-pages分支上。
``` bash
$ hexo generate
```
#### 3.本地仓库丢失
当你想在其他电脑工作，或电脑重装系统后，安装Git与Node.js后，可以使用下列步骤：
##### 1)拷贝仓库
``` bash
$ git clone https://github.com/mroccyen/myblog.git 
```
##### 2)配置Hexo
在本地新拷贝的myblog文件夹下通过Git bash依次执行下列指令:
``` bash
$ npm install -g hexo-cli
$ npm install hexo
$ npm install
$ npm install hexo-deployer-git --save
```

##### 3)小Tips:hexo 命令
``` bash
hexo new "postName" #新建文章
hexo new page "pageName" #新建页面
hexo generate #生成静态页面至public目录
hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy #将.deploy目录部署到GitHub
hexo help  # 查看帮助
hexo version  #查看Hexo的版本
hexo deploy -g  #生成加部署
hexo server -g  #生成加预览
#命令的简写
hexo n == hexo new
hexo g == hexo generate
hexo s == hexo server
hexo d == hexo deploy
```