---
layout: post
title:  "jekyll环境搭建"
date:   2019-12-15T16:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
catalog: true
tags:
    -  Life
---
# 前言
作为一名程序媛， 想搞个地方放自己整理的文件， 用过 csdn，或者本地的有道云笔记， 或者传到 github上， 但是感觉都不太高级。后来了解到可以通过github搭建的自己的博客网站， 甚是高级，值得搞一搞。

# 一波三折
我是一名基于linux平台得开发者，当然 使用linux系统要比windows要更舒服一些。所以一开始我就在linux上开始折腾，尝试了ubuntu18.0， 换了两个虚拟机， 都是各种问题， 某些gem包不能下载，jekyll一直下载不成功， 要么就是 下载成功了，*jekyll -v*也能显示版本号， 但是 *jekyll serve*使用时各种报错， 无奈以及无语。后来索性换了ubuntu20.04， 心想， 我都升级系统了， 总不会给我报错了吧，毕竟之前，18.04 默认安装ruby版本太低， 需要升级到20.04默认安装的ruby版本。但是， 天不随我愿， 还是一堆报错， 各种下载不了， 各种依赖问题。各种折腾之后，果断放弃。转战windows系统，由于本人在windows上使用不太熟悉， 先是在win7上， 遇到```ssl_connect returned=1 certificate verityu failed```报错，由于尝试的电脑是办公电脑，上边安装了很多的环境， 不敢随便动， 后来又转战自己打得一个服务器，用的是 win10, 安装成功。

# 安装
## 安装ruby
在Windows上使用RubyInstaller安装比较方便,去Ruby https://rubyinstaller.org/downloads/官网下载最新版本的RubyInstaller.注意32位和64位版本的区分. 注意：勾选添加到PATH选项,以便在命令行中使用。安装完成后会有一个安装msys2的界面，选择3， 回车。![安装msys2](/assets/doc/jekyll_build_msys2.png)

## 安装Rubygems
Windows中下载ZIP (下载页面 https://rubygems.org/pages/download)格式比较方便,下载后解压到任意路径.打开Windows的cmd界面,输入命令： $ cd {unzip-path} // unzip-path表示解压的路径, 然后执行```gem install jekyll```将会安装jekyll的一堆依赖，最后执行`jekyll -v`能成功。

## 使用
网上找一个自己想要的模板， 改吧改吧， 改成自己的风格， 然后进入目录， 执行`jekyll serve --host 0.0.0.0` , 然后通过浏览器就可以访问啦。

# *坚持不懈，没有什么解决不了的！*
