---
# 文章标题
title: "{{ replace .Name "-" " " | title }}"
# 文章内容摘要
description: "{{ .Name }}"

# 发表日期
date: {{ .Date }}
# 最后修改日期
lastmod: {{ .Date }}
# 分类
categories:
 - 
# 标签
tags:
 - 

#文章是否开启评论
comment:
  enable: true
# 用于对外访问的地址
url: artice/{{ .Name }}
# 是否显示目录
toc: true

# 文章封面图片相关属性
cover:
   image: "" #图片路径例如：posts/tech/123/123.png
   zoom: 50% # 图片大小，例如填写 50% 表示原图像的一半大小
   caption: "" #图片底部描述
   alt: ""
   relative: false

#是否为草稿
draft: false
---
