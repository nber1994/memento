---
title: 如何基于github pages搭建blog
date: 2017-12-06
categories: 
- tools
---
> 胡乱的记载了如何在github-pages搭建bolg

# 什么是github-pages

github推出pages服务时是如下所说：

> GitHub Pages allow you to publish web content to a github.com subdomain named after your username. With Pages, publishing web content becomes as easy as pushing to your GitHub repository.
If you create a repository named you.github.com, where you is your username, and push content to it, we’ll automatically publish that to http://you.github.com. No FTP, no scp, no rsync, nothing. Just a simple git push and you’re done. You can put anything here you like. Use it as a customizable home for your Git repos. Create a blog and spread your ideas. Whatever you want!


# 准备工作

* 首先你要有个github账号，添加自己电脑的公钥
* 创建自己的专门的仓库，专用来pages，命名格式为XXXX.github.io 
* 此时，你访问XXX.github.io即可访问自己的pages
* 如果嫌弃这加载速度，可以注册一个域名，cname解析到这个域名，这样访问速度会十分的快，因为dns缓存的原因，首次通过你自己购买的域名访问你的pages之后，下次访问会十分快，因为是直接通过国内dns服务器进行ip寻址


# 挑选主题

* 你可以自己写一个
* 也可以获取别人开源的框架（鉴于访问速度，建议选比较简洁的主题）
* search jekll theme 你会找到你想要的
* Next是一个不错的框架

## 挑选好之后，git clone 到本地，并添加远程仓库为你的XXX.github.io

# jekll 框架的目录结构

```shell
-rw-r--r--@  1 jingtianyou  staff   104B  9 18 07:47 404.html
-rw-r--r--@  1 jingtianyou  staff   1.0K  9 18 07:47 LICENSE
-rw-r--r--@  1 jingtianyou  staff   375B  9 18 07:47 README.md
-rw-r--r--   1 jingtianyou  staff   365B 12  7 18:32 _config.yml
drwxr-xr-x@  3 jingtianyou  staff   102B 12  7 17:53 _data
drwxr-xr-x@  4 jingtianyou  staff   136B 12  7 20:12 _includes
drwxr-xr-x@  5 jingtianyou  staff   170B 12  7 18:16 _layouts
drwxr-xr-x@ 15 jingtianyou  staff   510B 12  7 20:42 _posts
drwxr-xr-x@  4 jingtianyou  staff   136B 12  7 18:45 _sass
drwxr-xr-x  12 jingtianyou  staff   408B 12  7 20:31 _site
drwxrwxrwx@  6 jingtianyou  staff   204B 12  7 18:45 assets
-rw-r--r--@  1 jingtianyou  staff   219B  9 18 07:47 index.html
drwxr-xr-x@  3 jingtianyou  staff   102B 12  7 18:45 me
drwxr-xr-x@  6 jingtianyou  staff   204B  9 18 07:47 categories
```

## 404.html
404页面，同样你可以使用一些公益404页面，例如腾讯404.

## __config.yml
框架的配置文件,不同的框架的配置文件一般不一样，具体请参考框架文档

## __includes
一般存储页眉页脚的格式html，可进行客制化的定制

## __posts
你的博文都存放在这个目录下，命名方式为
> 2017-12-08-文章名.md

文章的开头一般是这样的，声明布局，标题，日期，tag等等,框架一般会自动帮你分类

```shell
---
title: InnoDB各类语句的加锁方式与应用
date: 2017-12-06
categories:
- mysql
categories: 
- lnmp
---
...
...
```
## __site
jekll运行后，会将你的文字翻译成html文件，存到该目录下

## assets
前端静态文件

# 安装本地jekll

jekll其实就是一个markdown ==> html 的翻译器，将你的markdown翻译成html

安装：

```shell
$ gem install jekyll
~ $ jekyll new my-awesome-site
~ $ cd my-awesome-site
~/my-awesome-site $ jekyll serve
# => Now browse to http://localhost:4000
```
* 安装过程我敢肯定一定会不成功
* 具体请自行百度
* jekll监听4000端口
本地运行jekyll服务之后，可以一边写，一边查看效果

```shell
 jingtianyou@jingtianyoudeMacBook-Air  ~/github/windows-95-master   master  jekyll server --future
Configuration file: /Users/jingtianyou/github/windows-95-master/_config.yml
            Source: /Users/jingtianyou/github/windows-95-master
       Destination: /Users/jingtianyou/github/windows-95-master/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
                    done in 0.868 seconds.
 Auto-regeneration: enabled for '/Users/jingtianyou/github/windows-95-master'
    Server address: http://127.0.0.1:4000//
  Server running... press ctrl-c to stop.
      Regenerating: 1 file(s) changed at 2017-12-07 20:27:57 ...done in 0.390267 seconds.
[2017-12-07 20:31:24] ERROR `/tag/lnmp/' not found.
      Regenerating: 1 file(s) changed at 2017-12-07 20:31:47 ...done in 0.328509 seconds.
[2017-12-07 20:31:53] ERROR `/20171207/如何基于github-pages搭建博客' not found.
[2017-12-07 20:31:55] ERROR `/tag/lnmp/' not found.
[2017-12-07 20:32:06] ERROR `/tag/lnmp/' not found.
      Regenerating: 1 file(s) changed at 2017-12-07 20:32:42 ...done in 0.408106 seconds.
[2017-12-07 20:32:44] ERROR `/tag/lnmp/' not found.
[2017-12-07 20:32:48] ERROR `/tag/lnmp/' not found.
      Regenerating: 1 file(s) changed at 2017-12-07 20:33:04 ...done in 0.546599 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:34:24 ...done in 0.41305 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:34:46 ...done in 0.285784 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:42:02 ...done in 0.353415 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:46:13 ...done in 0.333329 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:46:19 ...done in 0.293639 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:47:56 ...done in 0.314423 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:52:00 ...done in 0.335078 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:52:30 ...done in 0.429857 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:52:51 ...done in 0.287992 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:54:28 ...done in 0.330737 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 20:56:08 ...done in 0.328462 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 21:00:29 ...done in 0.342248 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 21:02:59 ...done in 0.343119 seconds.
      Regenerating: 1 file(s) changed at 2017-12-07 21:03:15 ...done in 0.27084 seconds.
```

# tips:
* 启动jekyll时，加上--future参数，不然你今天写的文章是显示不出来的
* 修改__posts下的文章，不用重启服务，会自动更新
* 修改__config.yml文件时，需要重启服务才生效

# Have Fun!
