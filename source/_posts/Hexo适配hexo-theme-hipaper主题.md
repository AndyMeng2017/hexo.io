layout: post
title: Hexo适配hexo-theme-hipaper主题
date: 2019-8-13 11:30:24
categories: blog
tags: [Hexo]

description: Hexo

photos:

 - https://upload-images.jianshu.io/upload_images/539247-0f218c1489b3e567.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240

### Hexo适配hexo-theme-hipaper主题

------

[hexo官方网站](https://hexo.io/zh-cn/docs/)

> - [Node.js](http://nodejs.org/) (Should be at least nodejs 6.9)
> - [Git](http://git-scm.com/)

#### 开始使用

本地环境安装完毕后执行命令`hexo version`，查看是否成功



<!-- more -->



```shell
$ hexo --version
hexo-cli: 2.0.0
os: Windows_NT 6.1.7601 win32 x64
http_parser: 2.8.0
node: 10.16.2
v8: 6.8.275.32-node.54
uv: 1.28.0
zlib: 1.2.11
brotli: 1.0.7
ares: 1.15.0
modules: 64
nghttp2: 1.34.0
napi: 4
openssl: 1.1.1c
icu: 64.2
unicode: 12.1
cldr: 35.1
tz: 2019a
```

#### 建站

这里可以理解为建立本地仓库环境

`folder`为指定的目录环境

```
$ hexo init <folder>
$ cd <folder>
$ npm install
```

新建完成后，指定文件夹的目录如下：

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```



更多命令见官方文档，挺全的

#### 获取指定的主题

##### Get it from Github

在你**建站**指定的 `folder`目录

`$ git clone https://github.com/iTimeTraveler/hexo-theme-hipaper.git themes/hipaper`

##### Enable 启用主题

修改配置文件`_config.yml`的主题

```yml
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
#默认主题
#theme: landscape
theme: hipaper
```

##### Update 更新主题

```
$ cd themes/hipaper
$ git pull
```

#### 配置 

> 针对根目录配置文件修的修改

##### 部署

安装 [hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)

```
`$ npm install hexo-deployer-git --save`
```

修改配置文件`_config.yml`的`Deployment`

```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: https://github.com/AndyMeng2017/andymeng2017.github.io.git
  branch: master
```

##### 站点信息修改

修改配置文件`_config.yml`的`Site`

```
# Site
title: Andy Meng
subtitle:
description: How many roads must a man walk down.
keywords:
author: John Doe|Andy Meng
language: en
timezone:
```



------



> 针对主题目录下配置文件修的修改

##### logo图标

```
# Put your avatar.jpg into `hexo-site/themes/hipaper/source/` directory.
# url is target link (E.g. `url: https://hexo.io/logo.svg` or `url: css/images/mylogo.jpg`)
avatar: 
  enable: true
  width: 124
  height: 124
  bottom: 10
  #url: https://hexo.io/logo.svg
  url: css/images/mylogo.jpg
```

##### 社交信息

```
# Social Links
# Key is the name of FontAwsome icon.
# Value is the target link (E.g. GitHub: https://github.com/iTimeTraveler)
social:
  Github: https://github.com/AndyMeng2017
  Weibo: https://weibo.com/u/1962585291
  Twitter: 
  Facebook: 
  Google-plus: 
  Instagram: 
  Pinterest: 
  Flickr: 
  email: 707093428@qq.com
```

##### 评论

```
# comment ShortName, you can choose only ONE to display.
duoshuo_shortname: 
disqus_shortname: 
#livere_shortname: MTAyMC8yOTQ4MS82MDQ5
livere_shortname: MTAyMC80NTk0OC8yMjQ1OQ==
uyan_uid: 
wumii: 
```

##### 不蒜子

```
# Miscellaneous
google_analytics:
gauges_analytics:
baidu_analytics:
tencent_analytics:
busuanzi_analytics: true
twitter:
google_plus:
fb_admins:
fb_app_id:
```

模板里域名过期，需要修改

`themes\hipaper\layout\_partial\busuanzi-analytics.ejs`文件

```javascript
<% if (theme.busuanzi_analytics){ %>
	<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
	</script>
<% }%>
```

##### CNZZ

```
# CNZZ count
# cnzz_siteid: 1260716016
cnzz_siteid: 1277912853
```

这里很尴尬的说，主题作者可能忘记删掉一些代码，故修改

themes\hipaper\layout\_partial\after-footer.ejs`文件，删掉一下信息

```javascript
<div style="display: none;">
  <script src="https://s11.cnzz.com/z_stat.php?id=1260716016&web_id=1260716016" language="JavaScript"></script>
</div>
```



#### 发布部署

在根目录执行 `hexo g -d`



附件：

评论插件截图，这软件很棒

![hexo-1](https://upload-images.jianshu.io/upload_images/539247-504a8fa0f87a4b24.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)