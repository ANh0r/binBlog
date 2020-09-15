---
title: 网站迁移啦
date: 2020-08-21 20:00:59
tags: Notice
cover: https://images.ryanhor.com/migr.png
---

# 迁移说明

经过一下午的曲曲折折，踩了不多不少的小坑，终于算是把`ts.ryanhor.com`的静态网站部署到了主站上！
其实我最近还是准备再github pages上进行一部分备份的，所以原来网站会同步进行部署，顺便可以体会下GitHub和直接走服务器的速度到底有什么不同！

## 迁移过程

* deploy changed
* Recover bt
* push
* done

## 后续处理：
 * hexo本地部署
 * 服务器上git处理
 * git hook files
 * ssh 免密登录push（秘钥连接）
 * 资源处理

 ## CDN

 * 七牛CDN图床建立
 * 加速域名绑定申请
 * 二级域名证书
 * 插入图床图片，实现客户端上传复制连接

 ## 绑定

 * 实现域名（包括不限于顶级域名，二级域名）与部署上来的public文件夹的绑定
 * 证书部署
 * hexo设置