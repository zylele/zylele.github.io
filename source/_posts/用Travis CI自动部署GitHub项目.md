---
title: 用Travis CI自动部署GitHub项目
date: 2017-09-10 23:19:05
tags: 
  - 持续集成
---

介绍一套免费持续集成构建部署解决方案。

![Poor Guy](https://zylele.github.io/img/travis-ci/day-day-poor.jpg)

本文以部署基于Node.js的静态博客框架hexo为例

源码地址：[zylele/blog](https://github.com/zylele/blog)

<!-- more -->

## Travis CI

顾名思义，Travis CI是一个持续集成(Continuous integration，简称CI)的工具。它可以在公共的Github仓库上免费使用。

> Travis CI 是目前新兴的开源持续集成构建项目，它与jenkins，GO的很明显的特别在于采用yaml格式，简洁清新独树一帜。目前大多数的github项目都已经移入到Travis CI的构建队列中，据说Travis CI每天运行超过4000次完整构建。

## 构建

### 在Github建立代码库

首先，要在Github上建立一个代码仓库，要将自己hexo博客push到上面。hexo项目作为运行部署的项目，然后Github Page的项目作为部署的目标项目。

### 开启Travis CI

第二步，我们需要有一个Travis CI的账号，直接进入[Travis CI](https://travis-ci.org/)官网，用自己的Github账号授权登录即可。

然后可以看到当前账号的所有代码仓库，接下来将博客项目的状态设置为启用。

![turn on travis](https://zylele.github.io/img/travis-ci/account.png)

### 创建SSH key

**如果你的github已经配置过SSH了，可以省略这一步，或者你想缩小权限控制粒度，可继续执行这一步骤。**

第三步，创建一个部署在Travis CI上面的SSH key利用这个SSH key可以让Travis CI向我们自己的项目提交代码。

```bash
$ ssh-keygen -t rsa -C "youremail@example.com"
```

得到`id_rsa.pub`和`id_rsa`，然后将有`pub`后缀的配置到Deploy key。

![set deploy key](https://zylele.github.io/img/travis-ci/deploy-key.png)

记得要将`Allow write access`的选项选上，这样Travis CI才能获得push代码的权限。

### 加密私钥

刚才讲公钥文件配置好了，然后就要配置私钥文件，在hexo项目下面建立一个`.travis`的文件夹来放置需要配置的文件。

首先要安装travis命令行工具(如果在国内的网络环境下建议安装之前先[换源](https://ruby.taobao.org/))。

```bash
$ gem install travis
```

用命令行工具登录：

```bash
$ travis login --auto
```

然后将刚刚生成的`id_rsa`复制到`.travis`文件夹，用命令行工具进行加密：

```bash
$ travis encrypt-file id_rsa --add
```

这个时候会生成加密之后的秘钥文件`id_rsa.enc`，原来的文件`id_rsa`就可以删掉了。

这时可以看到终端输出了一段

`openssl aes-256-cbc -K $encrypted_xxxxxxxxxxx_key -iv $encrypted_xxxxxxxxxxx_iv`

这样格式的信息，这是travis用来解密`id_rsa.enc`的key，先保存起来，后面配置`.travis.yml`会用到它。

为了让git默认连接SSH还要创建一个`ssh_config`文件。在`.travis`文件夹下创建一个`ssh_config`文件，输入以下内容：

```yml
Host github.com
    User git
    StrictHostKeyChecking no
    IdentityFile ~/.ssh/id_rsa
    IdentitiesOnly yes
```

现在进入travis CI设置页面

![travis setting](https://zylele.github.io/img/travis-ci/setting.png)

可以看到刚刚travis命令行生成的解密key

![environment variables](https://zylele.github.io/img/travis-ci/environment-variables.png)

顺便把上面的开关打开

![trun on travis setting](https://zylele.github.io/img/travis-ci/turn-on-build.png)

这样，当向项目push代码的时候travis CI就会根据`.travis.yml`的内容去部署我们的项目了。

### .travis.yml

最后就要配置`.travis.yml`。在项目的根目录创建`.travis.yml`文件。

```yml
# 配置语言及相应版本
language: node_js
sudo: false
node_js: stable
# 项目所在分支
branches:
  only:
  - master
# node_modules 缓存
cache:
  directories:
  - node_modules
# 配置环境
before_install:
# 替换为刚才生成的解密信息
- openssl aes-256-cbc -K $encrypted_xxxxxxxxx_key -iv $encrypted_xxxxxxxxx_iv -in .travis/id_rsa.enc -out ~/.ssh/id_rsa -d
# 改变文件权限
- chmod 600 ~/.ssh/id_rsa
# 配置 ssh
- eval $(ssh-agent)
- ssh-add ~/.ssh/id_rsa
- cp .travis/ssh_config ~/.ssh/config
# 配置 git 替换为自己的信息
- git config --global user.name 'zylele'
- git config --global user.email 657345933@qq.com
# 安装依赖
install:
- npm install
# 部署的命令
script:
- hexo clean
- hexo g -d
```

好了现在只要向项目push代码就可以触发部署了，进入[https://travis-ci.org](https://travis-ci.org)就可以看到部署的过程了。

## 后记

在部署了一遍之后发现，运行`npm install`安装node的库时候占据了部署的很大一部分时间，这里有一个技巧，可以将`node_modules`缓存起来，这样可以节省部署的时间。

```yml
# node_modules 缓存
cache:
  directories:
  - node_modules
```

## 最后

`.travis.yml`的完整代码可以看我的[.travis.yml](https://github.com/zylele/blog/blob/master/.travis.yml)文件。博客的完整代码可以看[这里](https://github.com/zylele/blog)。
