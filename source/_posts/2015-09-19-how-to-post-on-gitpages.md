---
layout: post
title:  "使用GitPage管理博客"
date:   2015-09-19 13:37:00
categories: how2use
---

**本文只介绍时下主流的几种工具，其他静态页面生成器的也可灵活运用，托管于GitPage**

# Jekyll

特点：官方推荐，Ruby环境

### 创建仓库

根据官方文档,

[https://pages.github.com/](https://pages.github.com/)

创建一个同名的仓库。


### Jekyll的使用

On Ubuntu,

    $sudo apt-get install ruby ruby-dev nodejs gem
    $sudo gem install jekyll
    
On Mac,

    $brew install ruby
    $gem install jekyll
    
创建blog,

    $jekyll new myblog
    $cd myblod
    $jekyll serve


可以在这里换主题,

[http://jekyllthemes.org/](http://jekyllthemes.org/)


`_posts` 就是放Markdown文章的目录,

    $mkdir _posts
    
文件名格式： `yyyy-mm-dd-postname.md`

```
2011-12-31-new-years-eve-is-awesome.md
2012-09-12-how-to-write-a-blog.textile
```

# Hexo

特点： 更丰富的主题，nodejs环境

### 安装nodejs

安装nodejs，并检查版本：

```
$ node -v  #(Should be at least nodejs 6.9)
$ npm -v
```

安装hexo-cli：

```
$ npm install -g hexo-cli
```

若在Mac上安装出现权限问题，使用以下步骤解决：

(有些资料写使用sudo，但在我的Mac不适用，查阅官方文档找到解决办法如下)

```
$ mkdir ~/.npm-global

$ npm config set prefix '~/.npm-global'

# 在~/.bash_profile文件中添加
$ export PATH=~/.npm-global/bin:$PATH

# 使之立即生效
$ source ~/.bash_profile

# 测试是否成功
$ npm install -g jshint

# 若成功则安装hexo-cli
$ npm install -g hexo-cli
```

### 使用hexo创建blog

初始化一个blog：

```
$ hexo init myblog
$ cd myblog
$ npm install
```

生成静态页面：

```
# 很万能的命令，各种问题出现先clean试下比较好，尤其是换了主题后
$ hexo clean

# 生成静态页面
$ hexo g
```

本地运行：

```
# 若端口被占用，使用-p 4001替换默认4000端口
$ hexo s

命令行提示访问URL，在浏览器输入即可预览
```

### 定制hexo

#### 推送GitHub

安装插件：

```
$ npm install hexo-deployer-git --save
```

使用推送的方式来发布blog，不要自己commit，修改myblog目录下的 `_config.yml` 配置:

```
deploy:
  type: git
  repo:
    github: https://github.com/yourname/yourname.github.io.git
  branch: master
```

`_config.yml` 中还有一些其他配置，可以自行定制。

使用以下命令来推送：

```
$ hexo clean
$ hexo g -d
```

#### 修改主题

可以在官网选择主题：https://hexo.io/themes/

比如最热门的next主题替换方法是：

```
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```

配置 `_config.yml` 的主题选项： `theme: next` 即可。

在主题 `themes/next/_config.yml` 配置中也可以定制个性化选项。

# Hugo

特点： 轻巧纯净，不依赖环境

使用简单，看[官方文档](https://www.gohugo.org/)即可搞定：建立blog，切换主题，推送gitpage的步骤。



# markdown文件编辑

blog最重要的当然是内容，推荐平时记笔记用md文件记录，不管你使用什么工具和主题，切换都很方便啦。

解析 `.md` 规则：

[https://guides.github.com/features/mastering-markdown/](https://guides.github.com/features/mastering-markdown/)

在线编辑工具,
 
[http://dillinger.io/](http://dillinger.io/)

Mac上的编辑工具： `Macdown` `typora`


# 参考

https://hexo.io/zh-cn/docs/index.html

https://docs.npmjs.com/resolving-eacces-permissions-errors-when-installing-packages-globally

https://www.gohugo.org/