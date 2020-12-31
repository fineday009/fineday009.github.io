---
title: 用hexo + git-pages + travis-ci搭建个人博客实践
date: 2020-12-29 23:25:00
author: ice
img: /source/images/xxx.jpg
top: true
cover: true
coverImg: /images/1.jpg
password: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
toc: true
mathjax: false
summary: 利用hexo实现博客壳子，利用git-pages外网托管博客，利用travis-ci实现了本地git push后博客分钟级自动部署。
categories: hexo
tags:
  - hexo
  - git-pages
  - travis-ci
---

## Step 1 of 4 利用hexo初始化博客

### 安装npm
- npm也就是node package manager，是node.js的依赖包管理工具，安装node后就自带了。
- windows和mac都可以直接从node.js官网下载包，然后本地解压安装，官网：https://nodejs.org/en/，一般安装最新LTS版就好。当然也可以用一些包管理工具安装，例如mac下使用以下命令即可秒装node.js。
``` bash
brew install node
```
输入以下命令验证node.js是否安装成功
```
❯ node -v
v14.15.3
```

### 安装hexo
原封不动，照抄官网的命令即可。
```bash
npm install hexo-cli -g
hexo init blog
cd blog
npm install
hexo server
```
上边的命令是利用npm下载了一个hexo的client，并利用hexo工具初始化一个博客，名为blog。
然后进入blog目录，安装node.js依赖，最终调用hexo server命令启动本地的hexo服务。

如果顺利，将看到类似以下的文字，告诉你你的博客已经在本地4000端口启动了，使用http://localhost:4000在浏览器打开就能看到博客首页。
```
❯ hexo server
INFO  Validating config
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```
hexo 默认是landscape的主题，比较朴素，其实也够用了。但我还是喜欢花里胡哨一点，所以翻了一些博客和网站去找hexo的主题。

从我搜索的结果来看，网上的主题五花八门，很多人封装了自己的主题，并放在了hexo的官网：https://hexo.io/themes/。

我大概随机点了几十个主题，总感觉有不满意。偶然看到hexo-theme-matery的主题，吸引了我的注意，正好符合我的审美。

## Step 2 of 4 配置博客主题
实际就是将hexo-theme-matery这个主题，应用到我的博客上。这里最佳的方式，应该是官网的一个个配置，https://github.com/blinkfox/hexo-theme-matery。
直接follow后，就能完成。

其中有一些插件和步骤不一定要做，取决于你的需要。例如我不需要RSS，这时候不安装rss插件即可。附我最后的效果图
![本地博客接入主题](/images/iceelocalblog-with-theme.png)


## Step 3 of 4 接入git-pages
这一步是想在外网通过域名直接访问我的博客，而不仅仅是通过http://localhost:4000这种方式。接入git-pages主要做2件事，新建github仓库并与本地的博客内容关联(git init)、在github页面上配置好git-pages。

我们将本地博客代码git push到github，通常过几分钟就能访问。例如我的blog：fineday009.github.io就是这样。

但目前又面临一个情况：每次本地修改完，需要调用hexo的命令去专门部署。能不能做成git push后，什么都不用管，坐等博客自动更新好。travis-ci就能完成这样的效果。

## Step 4 of 4 接入travis-ci
这里要做2件事，travis-ci绑定github的博客仓库、博客根目录配置.travis-ci.yml文件。
第一步实现效果如图。

第二步直接参考下边我的博客的实际例子，不用修改。需要注意GH_TOKEN这个名字是一个自定义名字，这个名字对应的value是一个token，需要在github.com生成。
``` travis-ci
sudo: false
language: node_js
node_js:
  - "12"
cache: npm
branches:
  only: 
    - master 
# build master branch only
script:
  - hexo generate
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: true
  on:
    branch: master
  local-dir: public
```
![GH_TOKEN配置](/images/travis-ci-fineday009.png)


## PS：配置评论区gittalk
- 编辑themes/hexo-theme-matery/_config.yml，增加如下配置。owner和admin都填github的账号，repo是为了存放评论而建的github的新仓库名，oauth的clientid和secret需要在github生成。
```
gitalk:
  enable: true #默认的是false，没有打开
  owner: 'fineday009'
  repo: 'fineday009.github.io.comments'
  oauth:
    clientId: 'your clientId'
    clientSecret: 'your clientSecret'
  admin: ['fineday009']
```