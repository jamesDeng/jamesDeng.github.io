---
layout: article
key: github_blog_install
tags: blog
comment: true
modify_date: 2018-6-28 10:00:00
---
# GitHub Blog
忽然发现写blog很好，可以记录自己的成长过程，采用GitHub Pages方式构建blog，不用服务器感觉真好！
## 选择一个模版
Jekyll是Github默认的模板系统,你可以在[Jekyll Themes](http://jekyllthemes.org/)找到你喜欢的模版fork下来，进行一系列配制就好了。
博主选择的是 [jekyll-TeXt](https://tianqi.name/jekyll-TeXt-theme/),后面操作都基于该模版。
## 设置 Repository Settings
1. 设置Repository name为"{自己的githup name}.githup.io",设置成功后就可以在浏览输入Repository name访问自己blog拉；![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/github_blog/Settings1.png)
2. 设置开启Issues ，后面评论要用;![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/github_blog/Settings2.png)
## 网站配制
找到网站根目录下的_config.yml，可以进行相关配制
### 基础配制
1. Site Settings
```YAML
text_color_theme: default #模版颜色 "default" (default), "dark", "forest", "ocean", "chocolate", "orange"
url: # the base hostname & protocol for your site e.g. https://www.someone.com
baseurl: # does not include hostname
title: 标题
description: > # this means to ignore newlines until "Language & timezone"
  blog 描述
```
2. 语言和时区
```YAML
lang: zh # the language of your site, "en" (default, English), "zh"(简体中文), "zh-Hans"(简体中文), "zh-Hant"(繁體中文)
timezone: # see https://en.wikipedia.org/wiki/List_of_tz_database_time_zones for the available values
```
3.作者信息
```YAML
author:
  name: 邓伟
  email: dengwei@thinker.vc # your Email address e.g. someone@site.com
  facebook: # your Facebook username
  twitter: # your Twitter username
  github: jamesDeng # your GitHub username
  linkedin: # your Linkedin username
  googleplus: # your Google+ username
  weibo: # your Weibo username
  douban: # your Douban username
```
4.GitHub 源码仓库，不是很理解文档上的说明，我配制成自己githup Repository,其格式为 项目所有者 ID / 项目名称
```YAML
repository: jamesDeng/jamesDeng.github.io
repository_tree: master
```
### 评论
为了没有墙的问题，采用的的gitalk，首先需要一个 GitHub Application，[点击这里申请](https://github.com/settings/applications/new),然后将相应的参数添加到 _config.yml 配置中
![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/github_blog/githuba_app_apply1.png)
![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/github_blog/githuba_app_apply2.png)

```YAML
gitalk:
  clientID:  # GitHub Application Client ID
  clientSecret:  # GitHub Application Client Secret
  repository: xxxx.github.io # GitHub repo
  owner: xxxx # GitHub repo owner
  admin:  # GitHub repo owner and collaborators, only these guys can initialize GitHub issues, IT IS A LIST.
    - "xxxx"
    # - your GitHub Id
```

这里需要注意的是gitalk是基于github Issues的，所以前面有提示项目Issues必须是打开的，然后你需要在页面的头信息里设置 key 属性来开启该页的评论。
### 点击量
TeXt 使用 LeanCloud 作为点击量功能的后台服务。你需要建立一个应用，然后在应用中建立一个 Class，之后将必要的信息填写到 _config.yml 文件中。下面详细介绍其操作步骤。

[点击这里](https://leancloud.cn/)进行主页，点击免费试用进行注册和登录，点击创建应用选择开发版
![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/github_blog/leancloud1.png)
点击应用上的存储进行存储管理页面，点击创建Class，设置数据条目的ACL 权限为无限制
![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/github_blog/leancloud2.png)
最后点击应用面板右侧的“设置”，点击“应用 Key” 选项，即可得到对应的 APP ID 和 APP KEY：
![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/github_blog/leancloud3.png)
![](https://raw.githubusercontent.com/jamesDeng/jamesDeng.github.io/master/images/github_blog/leancloud4.png)
```YAML
leancloud:
  app_id: kN1v8C5Og8KFAqD9xFpnpbRW-gzGzoHsz # LeanCloud App id
  app_key: z6gbTJXiyBeAtFz87iluVvLA # LeanCloud App key
  app_class: click_num # LeanCloud App class
```
## 文章
### 创建文章
发表一篇新文章，你所需要做的就是在 _posts 文件夹中创建一个新的文件。文件名的命名非常重要。Jekyll 要求一篇文章的文件名遵循下面的格式：
```YAML
年-月-日-标题.MARKUP
```
### YAML 头信息
```YAML
---
layout: article #布局，具体请看官方文档
key: k8s_aliyun_install #唯一标识文章，评论和点击量都需要用到
category: k8s
tags: k8s  #标签，用于归档的时候给文章分类
title: k8s 阿里云安装
modify_date: 2018-6-1 18:13:00
---
```
### 代码高亮
jekyll的代码高亮需要使用如下语法
```Javascript
{ % highlight javascript % }
..代码..
{% endhighlight %}
```
个人感觉这样很不友好，在githup上根本没法看，正在找一种方式让blog支持原生的Markdown代码块写法
参考文档
=
[TeXt Theme 官方文档](https://tianqi.name/jekyll-TeXt-theme/docs/zh/quick-start)